# Python Houndify SDK

The Houndify Python SDK allows you to make streaming voice and text queries to the Houndify API from your Python project. The SDK provides two classes **TextHoundClient** and **StreamingHoundClient** for making text and voice queries to the Houndify API. See the *Usage* section below for more information.


## Changes in v2.0.0

* Both **TextHoundClient** and **StreamingHoundClient** raise a **HoundifySDKError** exception when authenticaion fails or connection to the server cannot be established. 
* *enableVAD* argument of **StreamingHoundClient** is False by default.


## Installation

```shell
pip install houndify
```


## Usage

The houndify package contains three classes:

* **TextHoundClient**
* **StreamingHoundClient**
* **HoundListener**


### Text Queries: TextHoundClient

This class is used for making text queries to the Houndify API. The constructor expects:

* `clientID`: This is the Client ID of your Houndify app.
* `clientKey`: This is the Client Key of your Houndify app.
* `userID`: This parameter should be unique for every user that uses your app.
* `requestInfo`: (optional) This should be a dictionary that will make the JSON response more accurate. See [RequestInfo](https://docs.houndify.com/reference/RequestInfo) for supported keys.
* `proxyHost`: (optional) Pass the host for HTTP Connect Tunnelling to make queries through proxy.
* `proxyPort`: (optional) Port for HTTP Connect Tunnelling.
* `proxyHeaders`: (optional) Extra headers for HTTP Connect Tunnelling.

```python
import houndify

clientId = "YOUR_CLIENT_ID"
clientKey = "YOUR_CLIENT_KEY"
userId = "test_user"
requestInfo = {
  "Latitude": 37.388309,
  "Longitude": -121.973968
}

client = houndify.TextHoundClient(clientId, clientKey, userId, requestInfo)
```

The method that is used for sending a text query is **TextHoundClient.query(query_string)** and it returns a dictionary with [response JSON](https://docs.houndify.com/reference/HoundServer) in it.

```python
response = client.query("What is the weather like today?")
```

You can reset [RequestInfo fields](https://docs.houndify.com/reference/RequestInfo) via the **TextHoundClient.setHoundRequestInfo(key, value)** and **TextHoundClient.removeHoundRequestInfo(key)** methods.

```python
client.setHoundRequestInfo("City", "Santa Clara")
client.removeHoundRequestInfo("City")
```


### Audio Queries: StreamingHoundClient

In order to send an audio query to the Houndify API you need to initialize a **StreamingHoundClient**. The constructor expects the following arguments:

* `clientID`: This is the Client ID of your Houndify app.
* `clientKey`: This is the Client Key of your Houndify app.
* `userID`: This parameter should be unique for every user that uses your app.
* `requestInfo`: (optional) This should be a dictionary that will make the JSON response more accurate. See [RequestInfo](https://docs.houndify.com/reference/RequestInfo) for supported keys.
* `sampleRate`: (optional) The sample rate of the audio, either 8000 or 16000 Hz. It will default to 16000 if not supplied.
* `enableVAD`: (optional) Boolean specifying whether you want to support Voice Activity Detection. If True *fill()* method will return True and ignore any more audio bytes after server detects silence. The default value is False.
* `useSpeex`: (optional) This flag enables Speex conversion of audio. The default value is False. See *Speex Compression* section below for instruction on setting Speex up.
* `proxyHost`: (optional) Pass host for HTTP Connect Tunnelling to make queries through proxy.
* `proxyPort`: (optional) Port for HTTP Connect Tunnelling.
* `proxyHeaders`: (optional) Extra headers for HTTP Connect Tunnelling.

```python
import houndify

clientId = "YOUR_CLIENT_ID"
clientKey = "YOUR_CLIENT_KEY"
userId = "test_user"
client = houndify.StreamingHoundClient(clientId, clientKey, userId, sampleRate=8000)
```

Additionally you'll need to extend and create **HoundListener**:

```python
class MyListener(houndify.HoundListener):
  #  HoundListener is an abstract base class that defines the callbacks
  #  that can be received while streaming speech to the server
  def onPartialTranscript(self, transcript):
    #  onPartialTranscript() is fired when the server has sent a partial transcript
    #  in live transcription mode.  'transcript' is a string with the partial transcript
    print("Partial transcript: " + transcript)

  def onFinalResponse(self, response):
    #  onFinalResponse() is fired when the server has completed processing the query and has a response.
    #  'response' is the JSON object (as a Python dict) which the server sends back.
    print("Final response: " + str(response))

  def onError(self, err):
    #  onError() is fired if there is an error interacting with the server.  It contains
    #  the parsed JSON from the server.
    print("Error " + str(err))
```


### Audio Queries: Code Example

Finally, you should start a request with passing an instance of your Listener to **StreamingHoundClient.start(listener)**, pipe the audio through **StreamingHoundClient.fill(samples)** and call **StreamingHoundClient.finish()** when the request is done.

*Note: StreamingHoundClient supports 8/16 kHz mono 16-bit little-endian PCM samples.*

```python
class MyListener(houndify.HoundListener):
  def onPartialTranscript(self, transcript):
    print("Partial transcript: " + transcript)

  def onFinalResponse(self, response):
    print("Final response: " + str(response))

  def onError(self, err):
    print("Error " + str(err))


client.start(MyListener())
# 'samples' is the list of mono 16-bit little-endian PCM samples
client.fill(samples)
result = client.finish() # result is either final response or error
```

There are four more useful methods in **StreamingHoundClient** for resetting sample rate, location and request info fields.

```python
client.setSampleRate(8000) #or 16000
client.setLocation(37.388309, -121.973968)
client.setHoundRequestInfo("City", "Santa Clara")
client.removeHoundRequestInfo("City")
```


### Conversation State

Houndify domains can use context to enable a conversational user interaction. For example, users can say "show me coffee shops near me", "which ones have wifi?", "sort by rating", "navigate to the first one". Both **TextHoundClient** and **StreamingHoundClient** have **setConversationState(state)** method that you can use to set [Conversation State](https://docs.houndify.com/reference/CommandResult#field_ConversationState) extracted from a previous response.

```python
client = houndify.TextHoundClient(clientId, clientKey, userId)

response = client.query("What is the weather like in Toronto?")

conversationState = response["AllResults"][0]["ConversationState"]
client.setConversationState(conversationState)

response = client.query("What about Santa Clara?")
```



## Speex Compression

You can use *pySHSpeex* module included in the SDK for sending compressed audio.

You have to pass the additional **useSpeex=True** flag to the StreamingHoundClient in order to enable compression.

```python
import houndify

clientId = "YOUR_CLIENT_ID"
clientKey = "YOUR_CLIENT_KEY"
userId = "test_user"
client = houndify.StreamingHoundClient(clientId, clientKey, userId, useSpeex=True)
```
