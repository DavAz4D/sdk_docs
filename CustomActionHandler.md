
# CustomActionHandler

When respond to an action using database method [On Mobile App Action](https://doc.4d.com/4Dv18R3/4D/18-R3/On-Mobile-App-Action-database-method.301-4901671.en.html), you could provide some data to the iOS app with the result($0) object

You could for instance display a message using `statusText`

```4d
$0:=New object("success"; True; "statusText"; "My information message to display")
```

```4d
$0:=New object("success"; False; "statusText"; "My warning/error message to display")
```

## But if you want to do more and the SDK do not provide it functionnality? (18R5, maybe 18.3)

For that you must provide a swift code, the action handler, which implement the protocol `ActionResultHandler`.

There is two way to do it, implement in `AppDelegate` or using an injected [`ApplicationService`](ApplicationService.md)

### Simple example

```swift
import QMobileAPI
import QMobileUI
import SwiftyJSON

extension AppDelegate: ActionResultHandler {
   func handle(result: ActionResult, for action: Action, from: ActionUI, in context: ActionContext) -> Bool {
        let json = result.json
        if json["doSomething"].boolValue {
            // if response containts "doSomething": true 
            // your code here
            return true // yes I handle the action
        }
        return false // no I do not care about the action
    }
}
```

in db method

```4d
$0:=New object("success";True;"doSomething";True)
```

### Full example 

To make polling on url and display a message when finishing

#### iOS side 

```swift
import SwiftyJSON
import Alamofire
import SwiftMessages

import QMobileAPI
import QMobileUI

extension AppDelegate: ActionResultHandler {

    fileprivate func doNetworkRequest(_ urlString: String, _ step: TimeInterval) {
        /*data request */_ = AF.request(urlString).responseJSON { response in
            guard let data = response.data, let json = try? JSON(data: data) else {
                logger.warning("Failed to decode JSON data from \(urlString)")
                return
            }
            let state = json["state"].stringValue
            switch state {
            case "InProgress", "Scheduled":
                let checkedStep = step <= 0 ? 60: step // default 60s if not defined
                DispatchQueue.userInitiated.after(checkedStep) { [unowned self] in
                    self.doNetworkRequest(urlString, step)
                }
            case "Succeeded", "Failed", "Cancelled":
                logger.debug("Polling \(urlString) end with state \(state)")
            default:
                logger.debug("Unknown state \(state) for polling result")
            }
            if let message = json["statusText"].string {
                if json["success"].boolValue {
                    SwiftMessages.info(message)
                } else {
                    SwiftMessages.warning(message)
                }
            }
        }
    }

    func handle(result: ActionResult, for action: Action, from: ActionUI, in context: ActionContext) -> Bool {
        let pollingParameters = result.json["polling"]
        if let urlString = pollingParameters["url"].string {
            doNetworkRequest(urlString, pollingParameters["step"].doubleValue)
            return true
        }
        return false
    }

}
```

### On db method

```4d
// ... some code to launch task and provide a task id 

$0:=New object("success";True;"polling";New object("url"; "http://myserver/4DAction/checkTaskFinish?id=XXX"))
```

### Endpoint 

for instance using 4d action

```4d
 // get variables to get task id
 ARRAY TEXT($anames;0)
 ARRAY TEXT($avalues;0)
 WEB GET VARIABLES($anames;$avalues)
 // TODO code to find in array the "id" var
 $taskId:= ...
 
 // code to find task information, stask etc
 C_OBJECT($result)
 $result:=...


// then send result
WEB SEND TEXT(JSON Stringify($result))
 
```

