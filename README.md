# Audio Transcription and Summarization

This Python application is designed to efficiently transcribe an audio file (m4a) and generate concise text summaries using Google Cloud Speech-to-Text and OpenAI's GPT-3.

## Purpose

This code is useful for scenarios where audio recordings require transcriptions and summaries of spoken content. It applies to podcasts, interviews, meetings, and any audio content.

## Prerequisites

Ensure that the following prerequisites are in place

1. **Google Cloud API Key**: A Google Cloud API key with access to the Speech-to-Text API is required. This key should be saved in a JSON file.

2. **OpenAI API Key**: An OpenAI API key to access OpenAI's GPT API.

3. **Docker**: Ensure Docker is installed on your system for building and running the containerized application.

## How to Build

```bash
meeting-summarizer/
├── Dockerfile
├── requirements.txt
├── app.py
├── audio.m4a
└── google-cloud-speech-API.json
```

### 1. Create a 'requirements.txt' file in the project directory. This file should list all the dependencies

```plaintext
requests==2.26.0
ffmpeg-python==0.2.0
google-cloud-speech==2.6.0
openai==0.27.0
```

### 2. Create a 'Dockerfile' in the project directory to build the Docker image
```
# Use the official Python image as the base image
FROM python:3.8-slim
# Install ffmpeg
RUN apt-get update && apt-get install -y ffmpeg
# Set the working directory in the container
WORKDIR /app
# Copy the requirements file into the container
COPY requirements.txt .
# Install the Python dependencies
RUN pip install --no-cache-dir -r requirements.txt
RUN pip install google-cloud-speech
RUN pip install google-cloud-storage
RUN pip install openai
# Copy the script into the container
COPY app.py /app/app.py
# Run the script and output the results to the current directory
CMD ["python", "app.py"]
```

### 3. Compose the 'app.py' script for transcribing and summarizing an M4A audio file by following these three steps:
* Convert it to WAV format
* Transcribe it using Google Cloud Speech-to-Text API
* Summarize it using OpenAI GPT-3
The resulting transcription and summary will be stored as text files in the same directory as the input file. An M4A audio file is required as input for this program. You can modify the 'app.py' code to customize parameters such as summary length, input and output file names.

```
from google.cloud import storage
from google.cloud import speech_v1p1beta1 as speech
from google.protobuf import duration_pb2
import os
import subprocess
import openai

# Google Cloud API key JSON file
key_file_path = 'google-cloud-speech-API.json'

# OpenAI API key
openai.api_key = 'OpenAI-API-key'

def authenticate_with_api_key(api_key_json_file):
    # Set the GOOGLE_APPLICATION_CREDENTIALS environment variable to use the JSON key file
    os.environ['GOOGLE_APPLICATION_CREDENTIALS'] = api_key_json_file

def convert_audio(m4a_file, wav_file):
    # Use ffmpeg to convert audio from m4a to wav
    subprocess.run(['ffmpeg', '-i', m4a_file, '-ar', '16000', wav_file])

def transcribe_wav_to_text(input_wav_file, output_text_file):
    # Initialize the Google Cloud Storage client
    storage_client = storage.Client()

    # Create a bucket
    bucket_name = ‘any-random-bucker-name’
    bucket = storage_client.create_bucket(bucket_name)

    try:
        # Upload the audio file to the bucket
        blob = bucket.blob('audio.wav')
        with open(input_wav_file, 'rb') as audio_file:
            blob.upload_from_file(audio_file)

        # Get the URI of the audio file
        gcs_uri = f"gs://{bucket_name}/audio.wav"

        # Initialize the Google Cloud Speech client
        client = speech.SpeechClient()

        # Configure the recognition request
        config = speech.RecognitionConfig(
            encoding=speech.RecognitionConfig.AudioEncoding.LINEAR16,
            sample_rate_hertz=16000,  # Sample rate
            language_code="en-US",  # Language code
        )

        audio = speech.RecognitionAudio(uri=gcs_uri)  # audio source using the URI

        # Perform the transcription
        response = client.long_running_recognize(config=config, audio=audio)

        # Get the transcription results
        operation_result = response.result(timeout=600)  # Adjust the timeout

        # Extract and store the transcribed text in the output text file
        transcribed_text = ""
        for result in operation_result.results:
            transcribed_text += result.alternatives[0].transcript + " "

        # Save the transcribed text to the output file
        with open(output_text_file, "w") as text_file:
            text_file.write(transcribed_text)

    finally:
        # Delete the bucket and its contents
        bucket.delete(force=True)

def summarize_text(input_text_file, output_summary_file):
    # Read the transcribed text from the input text file
    with open(input_text_file, "r") as text_file:
        transcribed_text = text_file.read()

    # Use ChatGPT to summarize the text
    response = openai.Completion.create(
        engine="text-davinci-003",
        prompt=f"Summarize the following text: {transcribed_text}",
        max_tokens=2000,  # Adjust the max_tokens for the desired summary length
    )

    # Extract the summary from the response
    summary = response.choices[0].text.strip()

    # Split the summary into bulleted points based on line breaks
    summary_lines = summary.split('\n')
    bulleted_summary = '\n'.join([f'- {line}' for line in summary_lines])

    # Save the bulleted summary to the output summary file
    with open(output_summary_file, "w") as summary_file:
        summary_file.write(bulleted_summary)

if __name__ == "__main__":
    input_m4a_file = "audio.m4a"  # name of the input M4A file
    transcribed_text_file = "transcribed_text.txt"  # Output text file name
    summary_file = "summary.txt"  # Output summary file name

    # Get the absolute path of the input M4A file
    script_directory = os.path.dirname(os.path.abspath(__file__))
    input_m4a_file = os.path.join(script_directory, input_m4a_file)

    # Get the absolute path of the output text and summary files
    transcribed_text_file = os.path.join(script_directory, transcribed_text_file)
    summary_file = os.path.join(script_directory, summary_file)

    # Authenticate with the API key
    authenticate_with_api_key(key_file_path)

    # Convert M4A to WAV and transcribe the audio file
    input_wav_file = "audio.wav"
    convert_audio(input_m4a_file, input_wav_file)
    transcribe_wav_to_text(input_wav_file, transcribed_text_file)

    print(f"Transcription saved to {transcribed_text_file}")

    # Summarize the transcribed text and save the summary
    summarize_text(transcribed_text_file, summary_file)
    print(f"Summary saved to {summary_file}")
```

### 4. Copy the audio file 'audio.m4a' in the same project directory 

### 5. Run the Docker Container 
From the terminal, navigate to the project directory containing the Dockerfile and other files, and build the Docker image using the command:
```
docker build -t meeting-summarizer-image .
```
Execute the Docker container using the following command to mount the current directory as a volume, enabling the output text file to appear in the same location; this command leverages $(pwd) to automatically identify the current directory and mount it as a volume within the container, executing app.py within it and granting access to any output files generated by the script in the current directory on the host system.
```
docker run -v "$(pwd):/app" -it meeting-summarizer-image
```

### 6. Retrieve Transcriptions and Summaries
The application will save the transcribed text in 'transcribed_text.txt' and the summarized text in 'summary.txt' in the directory where you executed the Docker container.
```
...
size=    7306kB time=00:03:53.79 bitrate= 256.0kbits/s speed=3.82e+03x    
video:0kB audio:7306kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: 0.001444%
Transcription saved to /app/transcribed_text.txt
Summary saved to /app/summary.txt
```

This README provides a detailed guide on how to set up and use the Audio Transcription and Summarization application in a Docker container. Adjust the paths, filenames, and API keys to match your use case.

Happy transcribing and summarizing!


## Additional explanation of app.py

This Python script showcases a workflow involving audio file transcribing and summarizing the transcribed text using Google Cloud and OpenAI services. Below, you'll find an explanation of the script's key components and steps:

* Imports: The script imports necessary libraries, including google.cloud for Google Cloud services, subprocess to run shell commands, and openai for OpenAI API access.
* API Key Configuration: It sets the path to the Google Cloud API key JSON file (key_file_path) and configures the OpenAI API key (openai.api_key) with your respective API keys. Replace 'google-cloud-speech-API.json' and 'OpenAI-API-key' with your actual API key file and OpenAI API key.
* Authentication: The function authenticate_with_api_key(api_key_json_file) sets the GOOGLE_APPLICATION_CREDENTIALS environment variable to use the provided JSON key file for Google Cloud authentication.
* Audio Conversion: The function convert_audio(m4a_file, wav_file) uses the ffmpeg tool to convert an input M4A audio file (m4a_file) to a WAV audio file (wav_file) with a sample rate of 16000 Hz.
* Transcription: The function transcribe_wav_to_text(input_wav_file, output_text_file) performs the following:
    * Initializes the Google Cloud Storage client.
    * Creates a Google Cloud Storage bucket (Replace 'any-random-google-cloud-bucket-name' with your desired bucket name).
    * Uploads the converted WAV audio file to the bucket.
    * Specifies the audio source using the Google Cloud Storage URI.
    * Initializes the Google Cloud Speech client.
    * Configures the recognition request with parameters like encoding, sample rate, and language.
    * Performs the transcription asynchronously (client.long_running_recognize()).
    * Extracts and stores the transcribed text in an output text file.
* Summarization:
    * The function summarize_text(input_text_file, output_summary_file) summarizes the transcribed text using OpenAI's GPT-3 model. It does the following:
    * Reads the transcribed text from the input text file.
    * Uses OpenAI's GPT-3 engine ("text-davinci-003") to generate a summary based on the input text.
    * Extracts the summary from the response and formats it as bulleted points.
    * Saves the summarized text in an output summary file.
* Main Execution:
    * The script's main block performs the following steps:
    * Specifies the input M4A audio file, transcribed text file, and summary file names.
    * Authenticates with the Google Cloud API key.
    * Converts the M4A audio to WAV format.
    * Transcribes the WAV audio to text and saves it in the transcribed text file.
    * Summarizes the transcribed text and saves the summary in the summary file.
    * Prints messages indicating where the transcription and summary files are saved.

This script integrates Google Cloud's Speech-to-Text service for audio transcription and OpenAI's GPT-3 model for text summarization. It provides a streamlined approach to transcribing and summarizing audio content, making it useful for various applications.
