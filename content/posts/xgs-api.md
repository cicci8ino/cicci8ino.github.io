---
date: '2024-11-06T20:07:15+01:00'
draft: false
title: 'Reversing the login mechanism of a cheap Zyxel managed network'
tags: ["switch", "zyxel", "network", "rabbithole", "xgs1210"]
---

{{< admonition warning "what am I doing?" >}}
I'm not a developer nor a web app expert. Please, forgive me for my train of thought
{{< /admonition >}}

Some months ago, I bought a nice Zyxel XGS1210 managed switch for 130-ish euros on Amazon Marketplace. 2x10G SFPs, 2x2.5G RJ45, 8x1G RJ45. Not too bad.

I always wanted to manage it via Terraform/Tofu, as I'm doing with my opnsense install. Unfortunately, there is no Terraform provider already available and no public documentation/API, so I thought 
> It would make sense to lose hours in reverse engineering the web page to create a new VLAN every once in a while, right?

Nope, it probably didn't. Follow me in this rabbit hole.

# Login mechanism for http

First, let's examine how the login mechanism works. If you go to the switch IP, you are greeted by a beautiful login page straight from the 90s.

![Login page](/images/xgs-api/zyxel.png)

There's an interesting get request executed here:

```url
cgi/get.cgi?cmd=home_loginInfo&dummy=1730661645538&bj4=46f21041e4c399c1cd141211b4b4f547
```
That gives back this JSON data
```json
{
	"data":	{
		"sys_first_login":	"0",
		"model_name":	"XGS1210-12",
		"logined":	"0",
		"modulus":	"XXXXXX\n"
	},
	"xsrfToken":	""
}
```
Let's first focus on the request URL. `cmd` looks like a command, `dummy` looks like a timestamp. What about `bj4`? In `js/fileload.js`, the code would look something like this:

```js
url = url + '&bj4=' + md5(url.split('?')[1]);
```

So it looks like it's simply computing the `md5` of part of the requested URL.
Let's double check it
```bash
echo -n 'cmd=home_loginInfo&dummy=1730661645538' | md5sum
46f21041e4c399c1cd141211b4b4f547
```
Yep.

Let's write a simple `go` function to automatically compute the hash on each request.

```go
func GetURL(baseURL string, basePath string, action string) string {
	params := url.Values{}
	params.Add("cmd", action)
	params.Add("dummy", fmt.Sprint(time.Now().UnixMilli()))
	url := fmt.Sprintf("%s%s?%s", baseURL, basePath, params.Encode())
	url = url + "?bj4=" + ComputeMD5(url)
	log.Debug().Msg(url)
	return url
}
```

Where `ComputeMD5(completePath string)`

```go
func ComputeMD5(completePath string) string {
	result := strings.Split(completePath, "?")
	hash := md5.Sum([]byte(result[1]))
	md5String := hex.EncodeToString(hash[:])
	return md5String
}
```

Let's go back to the response the `home_loginInfo` gave. The only intersting data is `modulus`. As soon as the SIGN IN button is clicked, a `POST` request to is raised with 
`/cgi/set.cgi?cmd=home_loginAuth` and the following JSON body.
```json
{
	"_ds=1&password=something&xsrfToken=8b8107364f45c181&_de=1": {}
}
```
That's a strange JSON but ok. It seems we have 4 parameters stored in there somehow.

- `_ds`
- `password`
- `xsrfToken`
- `_de`

I absolutely have no idea of what `_ds` and `_de` mean, but they are always `1`, so I'm going to hardcode them and I'm going to send them in the exact order (yes, it was needed, wasted some time on this). On the other hand, `password` and `xsrfToken` are much more interesting.

Looking at `login.html`, we can see that `xsrfToken` is generated, creating a 16 chars string by picking random hex digit. Let's generate a random token while initializing the client. I'm not sure why it's the client that is generating the CSRF token, though...

`password` is being generated in the `clickLogin()` function in `login.html`, using functions stored in `crypt/rsa.js` file. This file seems to be coming from [here](https://github.com/creationix/jsbn/blob/master/README.md). 

First, a public RSA key is created from `modulus` and the hardcoded exponent.

```js
...
`rsa.setPublic(data.modulus, '<number>');`
...
```

The password is then encrypted with that key. More specifically, the encrypt() function will pad the string first and then encrypt the data using [PKCS1 v1.5](https://en.wikipedia.org/wiki/PKCS_1)

{{< admonition warning "PKCS1 v1.5" >}}
PKCS1 v1.5 is vulnerable to [Padding Oracle Attack](https://en.wikipedia.org/wiki/Padding_oracle_attack). An attacker reading the encrypted password might be able to decrypt it if the switch gives explicit feedback when an encrypted password with invalid padding is received. Additional information can be found [stackexchange](https://crypto.stackexchange.com/questions/12688/can-you-explain-bleichenbachers-cca-attack-on-pkcs1-v1-5).

Btw, it's already on a clear text channel, who cares?
{{< /admonition >}}

```js
...
// PKCS#1 (type 2, random) pad input string s to n bytes, and return a bigint
function pkcs1pad2(s,n)
...
```

The encrypted password is then encoded in base64 and then [serialized](https://api.jquery.com/serialize/).

If authorized, the `loginAuth` endpoint will reply with the following body:

```json
{
	"status": "ok",
	"authId": "XXXX"
}
```

The browser will then use the `authId` to POST the following request to `/cgi/set.cgi?cmd=home_loginStatus`.

```json
{
	"_ds=1&authId=XXXX>&xsrfToken=0011223344556677&_de=1": {}
}
```

Bingo, we now have a session cookie, `HTTP_SESSID` we can use from now on for every request. 

{{< admonition warning " xsrfToken" >}}
It may seem that the `xsrfToken` is hardcoded somehow. In reality, the `xsrfToken` is received as part of `GET` request response. If a `POST` request is submitted without an updated `xsrfToken`, the webserver will return 200 status code but no action will be performed.
{{< /admonition >}}


# Code

Let's put everything together. First, let's start with the helper functions responsible for creating the public key and encrypting the password.

```go
func CreatePublicKey(modulusHex string, exponentInt string) (*rsa.PublicKey, error) {
	modulusBytes, err := hex.DecodeString(modulusHex)
	if err != nil {
		log.Fatal().Err(err).Msg("failed decoding modulus")
		return nil, err
	}
	modulus := new(big.Int).SetBytes(modulusBytes)

	exponent, err := strconv.ParseInt(exponentInt, 16, 64)

	if err != nil {
		log.Fatal().Err(err).Msg("cannot convert exponent")
	}

	pubKey := &rsa.PublicKey{
		N: modulus,
		E: int(exponent),
	}
	log.Debug().Interface("pubKey", pubKey).Msg("pubKey")

	return pubKey, nil
}
```

```go
func EncryptPassword(pubKey *rsa.PublicKey, password string) (string, error) {
	passwordBytes := []byte(password)
	encryptedBytes, err := rsa.EncryptPKCS1v15(rand.Reader, pubKey, passwordBytes)
	if err != nil {
		log.Fatal().Err(err).Msg("error in encryption")
		return "", err
	}
	encryptedString := fmt.Sprintf("%x", encryptedBytes)
	if len(encryptedString)%2 != 0 {
		encryptedString = "0" + encryptedString
		log.Debug().Msg("detected odd lenght")
	}
	encryptedBytes, err = hex.DecodeString(encryptedString)
	if err != nil {
		log.Fatal().Err(err).Msg("error in reencoding")
	}
	encryptedPassword := base64.StdEncoding.EncodeToString(encryptedBytes)
	log.Debug().Msg(fmt.Sprintf("base64 encoded encrypted password: %s", encryptedPassword))
	return encryptedPassword, nil
}
```

Create a struct for our client.

```go
type SwitchClient struct {
	BaseURL    string
	Password   string
	XSRFToken  string
	HTTPClient *http.Client
}
```

Then, init the `SwitchClient` struct and all the needed objects, such as the HTTP client with a cookie store, the RSA public key and the `xsrfToken`.

```go
func (s *SwitchClient) Init() {
	// TODO: specify CA or use system certificates
	transport := &http.Transport{
		TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
	}
	jar, _ := cookiejar.New(nil)
	s.HTTPClient = &http.Client{
		Jar:       jar,
		Transport: transport,
	}
	requestURL := utils.GetURL(s.BaseURL, utils.CGIGETPath, utils.LoginInfoCMD)

	var loginInfoDataResp rawdata.LoginInfo
	s.StoreParsedData(requestURL, &loginInfoDataResp)
	parsedURL, err := url.Parse(s.BaseURL)
	if err != nil {
		log.Fatal().Err(err).Msg("invalid URL")
	}
	if parsedURL.Scheme == "http" {
		modulus := loginInfoDataResp.Modulus[:len(loginInfoDataResp.Modulus)-1]
		log.Debug().Msg(modulus)
		publicKey, err := utils.CreatePublicKey(modulus, utils.HEXExponent)
		if err != nil {
			log.Fatal().Msg("can't set public key")
		}
		s.Password, err = utils.EncryptPassword(publicKey, s.Password)
		if err != nil {
			log.Fatal().Err(err).Msg("cannot encrypt password")
		}
	}
	s.XSRFToken, err = utils.GenXSRFToken()
	if err != nil {
		log.Fatal().Err(err).Msg("cannot generate xsrftoken")
	}
}
```

Create the struct needed to store the `authId`.

```go
type AuthResponse struct {
	Status string `json:"status"`
	AuthID string `json:"authId"`
}
```

Login,

```go
func (s *SwitchClient) Login() {
	passwordParams := url.Values{}
	passwordParams.Add("password", s.Password)
	requestURL := utils.GetURL(s.BaseURL, utils.CGISETPath, utils.LoginAuthCMD)
	bodyBytes, err := s.Post(requestURL, passwordParams.Encode())
	if err != nil {
		log.Fatal().Err(err).Msg("login post failed")
	}

	var authResponse rawdata.AuthResponse
	err = json.Unmarshal(bodyBytes, &authResponse)
	if err != nil {
		log.Fatal().Err(err).Msg("problem umarshaling login data")
	}
	s.GetSessionCookie(authResponse.AuthID)
}
```

and get the session cookie,

```go
func (s *SwitchClient) GetSessionCookie() {
	requestURL := utils.GetURL(s.BaseURL, utils.CGISETPath, utils.LoginStatusCMD)
	bodyString := fmt.Sprintf(`{"_ds=1&authId=%s&_de=1":{}}`, authID)
	jsonData := []byte(bodyString)
	req, err := http.NewRequest("POST", requestURL, bytes.NewBuffer(jsonData))
	if err != nil {
		log.Fatal().Err(err).Msg("cannot post request")
	}
	req.Header.Set("Content-Type", "application/json")
	resp, err := s.HTTPClient.Do(req)
	for _, c := range resp.Cookies() {
		log.Debug().Msgf("%s=%s", c.Name, c.Value)
	}
}
```

Let's declare some helper functions:

```go
func (s *SwitchClient) FetchVLANData() rawdata.VLANData {
	url := utils.GetURL(s.BaseURL, utils.CGIGETPath, utils.VLANListCMD)
	var vlanDataResp rawdata.VLANData
	s.StoreParsedData(url, &vlanDataResp)

	return vlanDataResp
}
```
```go
func (s *SwitchClient) StoreParsedData(url string, v any) error {
	body, _ := s.Get(url)
	var result map[string]interface{}
	err := json.Unmarshal(body, &result)

	if err != nil {
		log.Fatal().Err(err).Msg("error parsing xsrfToken")
	}
	xsrftoken, ok := result["xsrfToken"].(string)
	if ok {
		// storing xsrftoken, to be used for subsequent post call
		s.XSRFToken = xsrftoken
	}
	err = json.Unmarshal(body, v)
	if err != nil {
		log.Fatal().Err(err).Msg("error parsing data")
	}
	data, ok := result["data"].(any)
	marshaledData, _ := json.Marshal(data)
	if err != nil {
		log.Fatal().Err(err).Msg("error marshaling data field")
	}
	err = json.Unmarshal(marshaledData, v)
	log.Debug().Interface("raw_data", v).Msg("raw_data")
	return nil

}
```
```go
func (s *SwitchClient) Get(requestURL string) ([]byte, error) {
	resp, err := s.HTTPClient.Get(requestURL)
	if (err) != nil {
		log.Fatal().Err(err).Msg("error fetching data")
	}
	body, err := io.ReadAll(resp.Body)
	defer resp.Body.Close()
	if err != nil {
		log.Fatal().Err(err).Msg("error reading body")
	}
	if resp.StatusCode != 200 {
		log.Error().Msg("get request returned non 200 response code")
		return nil, fmt.Errorf("resp code %d", resp.StatusCode)
	}
	return body, nil
}
```

`StoreParsedData()` will send an HTTP request, get the body and marshal it, save the `xsrfToken` (to be used for the next request) and then extract the relevant data in a generic struct.

For VLAN, 

```go
type VLANData struct {
	PVIDs  []int         `json:"pvids"`
	QVLANs []interface{} `json:"qvlans"`
}

type VLANDataResp struct {
	Data VLANData `json:"data"`
}
```

Aaaaaand, here we go:

```go
func main() {
	zerolog.SetGlobalLevel(zerolog.DebugLevel)
	log.Logger = zerolog.New(os.Stderr).With().
		Timestamp().
		Caller().
		Logger()
	baseURL := os.Getenv("ZYXEL_HOSTNAME")
	password := os.Getenv("ZYXEL_PASSWORD")
	zyxelSwitch := xgsapi.SwitchClient{
		BaseURL:  baseURL,
		Password: password,
	}
	zyxelSwitch.Init()
	zyxelSwitch.Login()
	zyxelSwitch.FetchSystemData()
	zyxelSwitch.FetchLinkData()
	zyxelSwitch.FetchVLANData()
}
```

```bash
{"level":"debug","raw_data":{"data":{"pvids":[1,100,1,1,1,200,200,1,1,1,1,10,1,1,1,1,1],"qvlans":[[1,"0x1ff9d","0x800"],[2,"0x1f000","0x1f000"],[10,"0x1fb80","0x1fb80"],[20,"0x1fb80","0x1fb80"],[30,"0x1f381","0x1f381"],[40,"0x1f381","0x1f381"],[100,"0x1f382","0x1f380"],[200,"0x1fbe1","0x1fb81"],[300,"0x1f003","0x1f000"]]}}
```

PoC code can be found [here](https://github.com/cicci8ino/xgs-api).

# Wait, what about HTTPs connection?

If the switch management interface is reached on the `HTTPs` URL, the login mechanism is much easier, as the password is sent as is in the same JSON body to the same `/cgi/set.cgi?cmd=home_loginAuth` endpoint, with no additional encryption other than the channel itself.