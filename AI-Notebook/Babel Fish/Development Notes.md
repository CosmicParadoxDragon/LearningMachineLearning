Watsonx flan-ul2 model for translation.


Pre built outline of code (again), the repo contains this structure.
```
models
-stt/
-*
-tts/
-*
README.md
static/
-script.js
-style.css
templates/
-index.html
Dockerfile
requirements.txt
server.py
worker.py
```

First task is the return the index with `render_template('index.html')`

IBM has overriden the requirements for a API Key for Watson in these labs, as usual.

Another block of code is given, this appears to be authentication information (credentials, object, project_id) more imports 
```
1. `from ibm_watson_machine_learning.metanames import GenTextParamsMetaNames as GenParams`
2. `from ibm_watson_machine_learning.foundation_models.utils.enums import DecodingMethods`
```

TODO:
Investigate GenTextParamsMetaNames and Decoding Methods added

Parameters, as well using the GenParams import above.
```
parameters = {
	GenParams.DECODING_METHOD: DecodingMethods.GREEDY,
	GenParams.MIN_NEW_TOKENS: 1,
	GenParams.MAX_NEW_TOKENS: 1024
}
```

Finally the Model data structure again.  This is more familiar, but I may need to restructure this as I did for the last lab, as there was a chang in the API.
```
model = Model(
    model_id=model_id,
    params=parameters,
    credentials=credentials,
    project_id=project_id
)
```
This is an assembly of the previous code blocks.  I will try this without making any changes, before comparing to the previous lab for a refactor.

The first function that I will be actually working on is the watsonx_process_message function which will take in a prompt and pass it the Watsonx flan-ul2 API.  The SEND BUTTON.

More boiler plate code is given wholesale, not entirely sure what I am suppose to work on here.
```
def watsonx_process_message(user_message):
    # Set the prompt for Watsonx API
    prompt = f"""Respond to the query: ```{user_message}```"""
    response_text = model.generate_text(prompt=prompt)
    print("wastonx response:", response_text)
    return response_text
```

 The prompt given to the API is refined by including the word translate and focusing on the English to Spanish conversion.  This is done because it is already understood what the user wants if they are using this application.
 ```
prompt = f"""You are an assistant helping translate sentences from English into Spanish. Translate the query to Spanish: ```{user_message}```."""
```

The next function to 'update' is `speech_to_text` in `worker.py`.  This give more boilerplate code, with only copy and paste required.  Well two copy and pastes as the base_url variable inside the code is left blank, for a yet unknown reason.

This function sends the received speech to be translated of to a model that will generate the corresponding text. specifically the model `en-US_Multimedia` this is obviously assuming an English to Spanish translation still.

I understand most of this code.  We are setting up more API requests and getting a response back, printing the response for debugging. then returning the final text. 

There seems to be some anomalous assignment happening on in the middle of the function, with: `body = audio_binary`.

Maybe that will get used later.

The next step is going the other way with Watson Text-to-Speech.  We are given a curl to see all the available voice options.
`curl https://sn-watson-tts.labs.skills.network/text-to-speech/api/v1/voices`
And another to actually test the functionality of these models.
One for English.
```
curl "https://sn-watson-stt.labs.skills.network/text-to-speech/api/v1/synthesize" --header "Content-Type: application/json" --data '{"text":"Hello world"}' --header "Accept: audio/wav" --output output.wav
```
Another for Spanish.
```
curl "https://sn-watson-stt.labs.skills.network/text-to-speech/api/v1/synthesize?voice=es-LA_SofiaV3Voice" --header "Content-Type: application/json" --data '{"text":"Hola! Hoy es un dia muy bonito."}' --header "Accept: audio/mp3" --output hola.mp3
```

Then we come again to copy paste, and the same as before a function is pasted in and another url is pasted inside that.
```
def text_to_speech(text, voice=""):
```
API Block
```
	# Set up Watson Text-to-Speech HTTP Api url
	base_url = 'https://sn-watson-tts.labs.skills.network'
	api_url = base_url + '/text-to-speech/api/v1/synthesize?output=output_text.wav'
```
Check for provided information and append the default if not given.
```
	# Adding voice parameter in api_url if the user has selected a preferred voice
	if voice != "" and voice != "default":
	    api_url += "&voice=" + voice
```
Headers
```
	# Set the headers for our HTTP request
	headers = {
	    'Accept': 'audio/wav',
		'Content-Type': 'application/json',
	}
```
HTTP Body
```
	# Set the body of our HTTP request
	json_data = {
	    'text': text,
	}
```
Send the request get a response, return the result.
```
	# Send a HTTP Post reqeust to Watson Text-to-Speech Service
	response = requests.post(api_url, headers=headers, json=json_data)
	print('Text-to-Speech response:', response)
	return response.content
```

Now we have to update the `server.py` file.

Here we have to update the two routes that the message takes.
Speech to Text route.
```
@app.route('/speech-to-text', methods=['POST'])
def speech_to_text_route():
    print("processing Speech-to-Text")
    audio_binary = request.data # Get the user's speech from their request
    text = speech_to_text(audio_binary) # Call speech_to_text function to transcribe the speech

    # Return the response to user in JSON format
    response = app.response_class(
        response=json.dumps({'text': text}),
        status=200,
        mimetype='application/json'
    )
    print(response)
    print(response.data)
    return response
```

The processes message route is the accept the user request and voice preference and runs it through the already defined functions.
```
@app.route('/process-message', methods=['POST'])
def process_message_route():
    user_message = request.json['userMessage'] # Get user's message from their request
    print('user_message', user_message)

    voice = request.json['voice'] # Get user\'s preferred voice from their request
    print('voice', voice)

    # Call watsonx_process_message function to process the user's message and get a response back
    watsonx_response_text = watsonx_process_message(user_message)

    # Clean the response to remove any emptylines
    watsonx_response_text = os.linesep.join([s for s in watsonx_response_text.splitlines() if s])

    # Call our text_to_speech function to convert Watsonx Api's reponse to speech
    watsonx_response_speech = text_to_speech(watsonx_response_text, voice)

    # convert watsonx_response_speech to base64 string so it can be sent back in the JSON response
    watsonx_response_speech = base64.b64encode(watsonx_response_speech).decode('utf-8')

    # Send a JSON response back to the user containing their message\'s response both in text and speech formats
    response = app.response_class(
        response=json.dumps({"watsonxResponseText": watsonx_response_text, "watsonxResponseSpeech": watsonx_response_speech}),
        status=200,
        mimetype='application/json'
    )

    print(response)
    return response
```

So now that we have built everything together, they now say that it will work fine.

Surprise.  It doesn't.  We docker build and run, and get this output.

```
$ docker run -p 8000:8000 voice-translator-powered-by-watsonx

'`apikey` for IAM token is not provided in credentials for the client'
Traceback (most recent call last):
  File "/app/server.py", line 6, in <module>
    from worker import speech_to_text, text_to_speech, watsonx_process_message
  File "/app/worker.py", line 33, in <module>
    model = Model(
            ^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watson_machine_learning/foundation_models/model.py", line 82, in __init__
    ModelInference.__init__(self,
  File "/usr/local/lib/python3.11/site-packages/ibm_watson_machine_learning/foundation_models/inference/model_inference.py", line 130, in __init__
    self._client = APIClient(credentials, verify=verify) 
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watson_machine_learning/client.py", line 161, in __init__
    raise WMLClientError(Messages.get_message(message_id="apikey_not_provided"))
ibm_watson_machine_learning.wml_client_error.WMLClientError: '`apikey` for IAM token is not provided in credentials for the client'
```

Reading through this traceback, I can see that the `apikey` is not provided when it is excepted.  Now, a few things.  The documentation for this tutorial specifically said that apikey wasn't required.  Obviously I believe that this means that I don't need to provide an API key for the Watsonx platform.  That is something that I can't really interact with as I believe it is implemented into the back end where the tutorial is interacting Watsonx and not my code. 

I did have to make this change for the previous lab, that worked with Watsonx in a similar way.  So, I have some idea of how to fix this already.  However I want to make a more systematic approach to the problem.

So the first attempt that I will make at fixing this problem.  By model chaning model to add the `apikey` variable to the model.  Because the credentials section already has an apikey argument in the comments.
```
# Define the credentials 
credentials = {
    "url": "https://us-south.ml.cloud.ibm.com",
    "apikey": 'API_KEY'
}
```
This is the edited version to uncomment the line.

Rebuilding and rerunning.

```
$ docker run -p 8000:8000 voice-translator-powered-by-watsonx                                                                                                         
Error getting IAM Token.
Reason: <Response [400]>
Traceback (most recent call last):
  File "/app/server.py", line 6, in <module>
    from worker import speech_to_text, text_to_speech, watsonx_process_message
  File "/app/worker.py", line 33, in <module>
    model = Model(
            ^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watson_machine_learning/foundation_models/model.py", line 82, in __init__
    ModelInference.__init__(self,
  File "/usr/local/lib/python3.11/site-packages/ibm_watson_machine_learning/foundation_models/inference/model_inference.py", line 130, in __init__
    self._client = APIClient(credentials, verify=verify) 
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watson_machine_learning/client.py", line 367, in __init__
    self.service_instance = ServiceInstanceNewPlan(self)
                            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watson_machine_learning/instance_new_plan.py", line 41, in __init__
    self._client.wml_token = self._get_token()
                             ^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watson_machine_learning/instance_new_plan.py", line 187, in _get_token
    return self._create_token()
           ^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watson_machine_learning/instance_new_plan.py", line 215, in _create_token
    return self._get_IAM_token()
           ^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watson_machine_learning/instance_new_plan.py", line 276, in _get_IAM_token
    raise WMLClientError(u'Error getting IAM Token.', response)
ibm_watson_machine_learning.wml_client_error.WMLClientError: Error getting IAM Token.
Reason: <Response [400]>
```

So another more complicated error is introduced.  Although the root problem is again not getting an IAM Token. 

After checking the documentation for the [`Model` class] (https://ibm.github.io/watson-machine-learning-sdk/model.html#ibm_watson_machine_learning.foundation_models.Model).  The `credentials` needs an `apikey` from the example given.  So I am going to use that for now. 

I have found a new source of information, someone from the discussion board posted this https://ibm.github.io/watsonx-ai-python-sdk/migration_v1.html link.  There seems to be a new SDK 1.0 for Watsonx and Python.

After trying this new approach using the new credential Class, and importing from the new SDK.  Including updating the requirements file for to include the `ibm_watsonx_ai` module. Then updating the credentials like so,
```
# Define the credentials 
credentials = Credentials(
    url="https://us-south.ml.cloud.ibm.com",
    api_key= "API_KEY"
)
```

This does something.  But not much.

```
$ docker run -p 8000:8000 voice-translator-powered-by-watsonx                                                                                    

Error getting IAM Token.
Reason: <Response [400]>
Traceback (most recent call last):
  File "/app/server.py", line 6, in <module>
    from worker import speech_to_text, text_to_speech, watsonx_process_message
  File "/app/worker.py", line 34, in <module>
    model = Model(
            ^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watsonx_ai/foundation_models/model.py", line 90, in __init__
    ModelInference.__init__(
  File "/usr/local/lib/python3.11/site-packages/ibm_watsonx_ai/foundation_models/inference/model_inference.py", line 148, in __init__
    self._client = APIClient(credentials, verify=verify)
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watsonx_ai/client.py", line 343, in __init__
    self.service_instance: ServiceInstance = ServiceInstance(self)
                                             ^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watsonx_ai/service_instance.py", line 55, in __init__
    self._client.token = self._get_token()
                         ^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watsonx_ai/service_instance.py", line 220, in _get_token
    return self._create_token()
           ^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watsonx_ai/service_instance.py", line 256, in _create_token
    return self._get_IAM_token()
           ^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.11/site-packages/ibm_watsonx_ai/service_instance.py", line 329, in _get_IAM_token
    raise WMLClientError("Error getting IAM Token.", response)
ibm_watsonx_ai.wml_client_error.WMLClientError: Error getting IAM Token.
Reason: <Response [400]>
```

The output is different, yet we end with the same Error of not getting an IAM Token.  The only difference is the `ServiceInstance` function is in the stack and not the `ServiceInstanceNewPlan` function.