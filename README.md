# GoogleClassroomRE
Documentation on reverse engineered Google Classroom API 

## Common parameters
| Param | Value|
| - | - |
| `_reqid` | (any int, can be same across all requests) |
| `ch_id` | 0/1 (int, used for multiple account selection, use same value as displayed in browser |

## Data
Requests to the internal APIs are submitted with post data containing what seems to be a template of variables to return, and a CSRF token. This data differs per API endpoint as different possible datasets can be returned, so use browser network tab to sample each request.

Example data: (token and ids have been changed)
```
f.req: [[1000,null,1,0],[[[true,true,true,true,true,false,null,[true,true,true,null,true,true,true],true,true,true,true,true,true,[true],null,null,null,true,3],[true,true,true,true,true,true,[true],true,null,[true,true],true,true,null,true,[[true,true,[],[null,true]],true],null,false],[null,true],null,[true,true]],[[true,true,true,true,true,false,null,[true,true,true,null,true,true,true],true,true,true,true,true,true,[true],null,null,null,true,3]],[[true,true,true,true,true,false,null,[true,true,true,null,true,true,true],true,true,true,true,true,true,[true],null,null,null,true,3],[true,true,true,true,true,true,[true],true,null,[true,true],true,true,null,true,[[true,true,[],[null,true]],true],null,false],[true]],null,null,[[true,true,true,true,true,false,null,[true,true,true,null,true,true,true],true,true,true,true,true,true,[true],null,null,null,true,3]]],[[[],[["674647454344"]],[2,5],[2],[],[],[],[],[],null,null,[],null,[],[1]]]]

token: u84u9eutjiowpurgioujf9q80uf98ufiquiutr98tu89888d
```
## Headers
Along with the authentication cookies, the only other required header is

`Content-Type: application/x-www-form-urlencoded`

## Authentication
Authentication is done using 3 cookies:
- `SID`
- `HSID`
- `SSID`

Which can be copied from a browser for testing purposes

## Getting valid json from API
The APIs return the same 4 characters before the JSON data `)]}'`. This can be cut from the start of the string and then the response can be interpretted as JSON.

## API
| Use/Desc | URL | Params | Data | More info |
| - | - | - | - | - |
| List lessons | `https://classroom.google.com/u/{ch_id}/v8/querycourse?_reqid={req_id}&rt=j` | `_reqid` and `ch_id` | Yes |  |
| Get work | `https://classroom.google.com/u/{ch_id}/v8/streamitem/query?_reqid={req_id}&rt=j` | `_reqid` and `ch_id` | Yes | |
| Submit work / mark task as done | `https://classroom.google.com/u/{ch_id}/v8/streamitem/query?_reqid={req_id}&rt=j` | `_reqid` and `ch_id` | Yes |

### Work
1. Fetch work from endpoint `https://classroom.google.com/u/{ch_id}/v8/streamitem/query?_reqid={req_id}&rt=j`, cut first 4 characters then load the JSON object.
2. Get the list of work from the object - `workJSON[0][0][2]`
3. Loop through each object in the list, using set offsets to get specific data points

| Data point | Index (python format) |
| - | - |
| Base | `[1]` | 
| Base -> workid | `[0][0][0]` |
| Base -> classid | `[0][0][1][0]` |
| Base -> createdmilis | `[1][1]` |
| Base -> duemilis | `[1][0]` |
| Base -> name | `[0][5]` |
| Base -> description | `[0][22][1]` |



Direct link to page is generated using base64 encoded values for classid and workid:
``` python
f"https://classroom.google.com/u/{ch_id}/c/{urlsafe_b64encode(self.classid.encode()).decode()}/a/{urlsafe_b64encode(self.workid.encode()).decode()}/details"
```

#### Getting state of submission

Post to `"https://classroom.google.com/u/{ch_id}/v8/querysubmission?_reqid={req_id}&rt=j`, adding classids and workids to the list in the data template.

| Data point | Index (python format) |
| - | - |
| Base -> submissionState | `[20]` |
| Base -> submissionmilis | `[21]` |

`submissionState` is an enum:
| Value | Text |
| - | - |
| 1 | No due date / Due (If `duemilis` is set then assume due) |
| 2 | Missing |
| 3 | Submitted |
| 4 | Handed in late |
| 8 / 10 | Marked | 

``` python
self.substate = base[20]
match self.substate:
    case 1:
        if self.duemilis is None:
            self.state = "No due date"
        else:
            self.state = "Due"
        self.submitted = False
    case 2:
        self.state = "Missing"
        self.submitted = False
    case 3:
        self.state = "Submitted"
        self.submitted = True
    case 4:
        self.state = "Handed in late"
        self.submitted = True
    case 8 | 10:
        self.state = "Marked"
        self.submitted = True
    case _:
        self.state = str(self.substate)
if self.submitted:
    self.submilis = base[21]
    self.subtime = datetime.utcfromtimestamp(
        self.submilis/1000).strftime('%Y-%m-%d %H:%M:%S')
```
## Error detection
If an error occurs then data at `[0][0][2]` will be `er`

``` python
j = json.loads(page.text[4:])[0][0][2]
if page.text[0][0][0] == "er":
    return False
```


