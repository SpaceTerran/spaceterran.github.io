---
title: "Efficient Email Management with Python and Ollama: A Gmail Summarization Script"
author: isaac
date: 2024-07-14 07:00:00 -0700
categories: [AI and Machine Learning, Automation, Ollama]
tags: [Python, AI, Ollama, Email automation, Gmail API, OAuth2, Email summarization, Scripting, Automation]
render_with_liquid: false
toc: true
comments: true
image:
  path: /assets/img/myphotos/Gmail-with-Ollama/Gmail-Summarization-Script-with-Ollama.png
  alt: 
---


### Introduction

In this post, I demonstrate how to leverage a fairly simple Python script that utilizes the power of Ollama—a self-hosted AI system—along with the Gmail API to read and summarize emails. The script is configured to pull a specific number of emails, currently set to 10 in its default state. While I may not share the entirety of the final script in this post (feel free to let me know in the comments if you're interested in seeing more!), I will use this as a foundational function to develop additional features that fully automate my personal Gmail account. 

By spending 10-30 minutes a day sorting junk emails, even saving just half that time can reduce the mental load of context-switching, making it a win all around.

This example illustrates how you can harness a locally hosted and private AI through the Ollama project on [GitHub](https://github.com/ollama/ollama). I often remind others that they have a powerful assistant at their fingertips, ready to optimize tasks or empower them—just provide clear instructions! By creating a step-by-step series of prompts, you can efficiently complete any project. 

### Setting Up the Environment

#### API and Credential Configuration

Before running the script, ensure you have the following configurations in place:

**Note:** These instructions have been tested running on a Mac, using a separate Linux-based Ollama server hosted in my homelab. The model used is `gemma2:27b`. When working on scripts and prompt engineering, your mileage may vary and you may need to tweak configurations to ensure reliable execution. Give it a try with this model version [here](https://ollama.com/library/gemma2:27b/blobs/d7e4b00a7d7a) to ensure the most consistent experience with the script below.

1. **Gmail API Credentials**: Follow the official [Google API documentation](https://developers.google.com/gmail/api/quickstart/python) to create OAuth 2.0 credentials. Below, I have taken some screenshots that you can follow to create this connection if you would rather not read through all the documentation.

   >I have circled the areas you need to click, fill in, or select in each picture. Pay attention to the numbering; it will guide you through the steps in the correct order if the screenshot includes multiple steps. Additionally, I have added lettering for steps that consist of multiple screenshots, so you know the sequence to follow. Don't forget, you can click on each photo to enlarge it; this might make it easier to read.
   {: .info-tip }

   1. Navigate over to [Google Cloud Console](https://console.cloud.google.com/), then create a new project.
   <br>
     A ![Create New Project](/assets/img/myphotos/Gmail-with-Ollama/1_create_new_project.png){: width="400"}
     B ![Name Project](/assets/img/myphotos/Gmail-with-Ollama/2_name_project.png){: width="300"}
   <br>
   2. Once the project is created, you need to switch projects if this isn't the first project. Use the down arrow, then select your project. Once it's loaded, use the quick access section to navigate to APIs and Services.
   <br>
     C ![Select New Project](/assets/img/myphotos/Gmail-with-Ollama/3_select_project.png){: width="400"}
     D ![Select api](/assets/img/myphotos/Gmail-with-Ollama/4_select_api.png){: width="300"}
   <br>   
   3. Select "Enable Apps and Services," search for "Gmail," select "Gmail API" from the search results, let the screen refresh, click on "Gmail API" again, and finally click "Enable."
   <br>
     E ![Select New Project](/assets/img/myphotos/Gmail-with-Ollama/5_add_api.png){: width="350"}
     F ![Select api](/assets/img/myphotos/Gmail-with-Ollama/6_search_api.png){: width="350"}
   <br>
     G ![Select New Project](/assets/img/myphotos/Gmail-with-Ollama/7_select_gmail_api.png){: width="350"}
     H ![Select api](/assets/img/myphotos/Gmail-with-Ollama/8_enable_gmail_api.png){: width="350"}
   <br>
   4. Select "Create Credentials".
   <br>
     I ![Select New Project](/assets/img/myphotos/Gmail-with-Ollama/9_create_creds_button.png){: width="700"}
   <br>
   5. Select "User data," then click "Next." Name your application, select your email address, optionally set a logo, enter a support email address, and then click "Save and Continue." In the next section, search for "gmail.readonly" select the checkbox next to the result, and then click "Save and Continue."
   <br>
     J ![Select New Project](/assets/img/myphotos/Gmail-with-Ollama/10_create_creds_userdata.png){: width="350"}
     K ![Select api](/assets/img/myphotos/Gmail-with-Ollama/11_create_creds_nameapp.png){: width="350"}
   <br>
     L ![Select New Project](/assets/img/myphotos/Gmail-with-Ollama/12_create_creds_select_scope.png){: width="350"}
     M ![Select api](/assets/img/myphotos/Gmail-with-Ollama/13_create_creds_select_scope_save.png){: width="350"}
   <br>
   6. Now select "Web application" as the type, then name your application or leave it as the default. The most important part is to add a URI in the "Authorized Redirect URIs" field; be sure to enter `http://localhost:8080`. Ensure you download the credentials here—this file will be used as your `credentials.json`. Simply rename the file and add it to the same directory where you are running this Python script.
   <br>
     N ![Select New Project](/assets/img/myphotos/Gmail-with-Ollama/14_create_creds_app_type.png){: width="400"}
     O ![Select api](/assets/img/myphotos/Gmail-with-Ollama/15_create_creds_download.png){: width="300"}
   <br>
   
   >Important: Don't forget to rename the downloaded file to `credentials.json`.
   {: .prompt-warning }
   
   7. This is the final step; it's possible that this will already be selected as "Internal." However, to confirm, follow these steps: ensure it is listed as "Internal."
   <br>
     P ![Select New Project](/assets/img/myphotos/Gmail-with-Ollama/16_select_oauth.png){: width="400"}
     Q ![Select api](/assets/img/myphotos/Gmail-with-Ollama/17_oauth_internal.png){: width="300"}
   <br>
   <br>
   >If you need help with anything above, please reach out to me via the contact form on the left-hand side of the page, or drop a comment in the comments section below. I'm happy to help!
   {: .info-tip }
   <br>

2. **Ollama API**: Ensure that you have the Ollama server set up and running. Update the `API_URL` and `MODEL` variables accordingly in the script. I'm not going to cover this in detail here, as the walkthroughs located in these links should get you started. A couple of items to keep in mind: ensure that the Ollama server is set up to receive off-host connections. For more information, check out [this guide](https://github.com/ollama/ollama/blob/main/docs/faq.md#how-do-i-configure-ollama-server). The main walkthrough for installation can be found [here](https://github.com/ollama/ollama/blob/main/README.md#quickstart).

#### Environment Variables

It's crucial to store sensitive information securely. The script uses an environment variable `EMAIL_PASSWORD` for the email sending function. Set this variable in your system to avoid exposing your credentials in the code.

```sh
export EMAIL_PASSWORD='your-email-password'
```

### Script Breakdown

#### Imports and Configurations

The script begins with module imports and basic configuration. Key modules like `requests`, `smtplib`, and `googleapiclient` come into play. Ensure all the required packages are installed, including `requests`, `google-auth`, `google-api-python-client`, and `beautifulsoup4`.

>There are multiple ways to install Python on a Mac. Using [Brew](https://brew.sh/) or using [Rye](https://rye.astral.sh/guide/installation/), just ensure you have installed all of these dependencies. It should look something like this:
```bash
pip3 install requests google-auth google-api-python-client beautifulsoup4
```
{: .prompt-info }

#### Authentication

The function `authenticate_gmail` handles Gmail API authentication, utilizing OAuth 2.0. The generated credentials are saved in `token.json` for reuse. The process here is: once you run the script the first time, a browser window will launch or a URL will be placed in the console window. Complete the login and approval process in a web browser. This will create the `token.json` in the same directory as the Python script is run.

#### Fetching Emails

The `get_messages` function fetches up to 10 emails(or change the amount in the script) from your Gmail inbox using the Gmail API. This function is configured to return email metadata, which will be further processed.

#### Processing Email Content

Next, the script includes utilities like `html_to_text` to convert HTML email content to plain text, and `get_message_details` to extract details such as the subject, sender, recipient, and labels. This mainly helps to ensure complex emails can be interpreted correctly.

#### Summarization Using Ollama

The `summarize_text` function integrates with the Ollama API, providing email content and receiving summarized text. The function constructs a detailed prompt and retrieves the AI-generated summary via HTTP POST.

#### Sending Summaries via Email

Finally, the `send_email` function sends a consolidated summary email using `smtplib`. All extracted summaries are aggregated and sent to the recipient configured in the script. This only works with **Gmail** at this time. (Note: You could change the script to another email provider for sure, or even use the Google API for sending emails.). 
>Note, if you have MFA or a similar setup with your account, you will need to create an application password for this to work. Check out this link [here](https://support.google.com/accounts/answer/185833?hl=en) for more information about creating an application password.
{: .prompt-warning }

### Future Enhancements
Again, not covered here, as this may or may not become a series of blog posts as I build or tweak the current functionality.

#### Email Management

- **Automatic Categorization**: Expand the script to categorize emails based on the AI’s evaluation.
- **Email Moving/Deletion**: Add functionality to move emails to specific folders or delete them if deemed unnecessary.

#### Customization

- **Custom Instructions**: Modify the `CUSTOM_INSTRUCTIONS` to fine-tune what the AI should analyze and summarize.

### The Code
```python
import os.path
import base64
import smtplib
import requests
from google.oauth2.credentials import Credentials
from google.auth.transport.requests import Request
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from tenacity import retry, stop_after_attempt, wait_exponential
from bs4 import BeautifulSoup
import re

# Things to change or can be changed
API_URL = "http://192.168.1.5:11434/api/generate" # This is the URL to your Ollama server.
MODEL = "gemma2:27b"  # Model parameter for the API request
INBOX_MAX_RESULTS = 10 # How many emails to process

# Email addresses for the sender and recipient
FROM_EMAIL = "your.email.address@gmail.com"
TO_EMAIL = "your.email.address@gmail.com"

# Custom instructions for the Ollama API
# You can customize this to fit your specific use case
# However, the instructions should be clear and concise, use a new line for each instruction. Ex: "9. Check the email for any attachments."
# Must update line 129 to include the new instruction numbers.
CUSTOM_INSTRUCTIONS = (
    "Analyze the provided email according to the following instructions 1 - 8:\n\n"
    "1. Extract key details from the email: subject, sender, recipient, body, and labels.\n"
    "2. Make a determination on whether the email is promotional.\n"
    "3. Summarize the email body in 1-2 sentences.\n"
    "4. Evaluate and determine the importance level of the email.\n"
    "5. Determine if a response is required.\n"
    "6. Categorize any promotional emails.\n"
    "7. Do not provide any other comments, breakdown, key elements, other information, marketing tactics used, HTML code snippet representation, etc.\n"
    "8. Always Format the response according to the specified: Example Format.\n\n"
    "Example Format:\n\n"
    "Subject: [Subject]\n"
    "Sender: [Sender]\n"
    "Recipient: [Recipient]\n"
    "Labels: [Labels]\n"
    "Promotional: [Yes/No]\n"
    "Response Required: [Yes/No]\n"
    "Importance: [Low/Medium/High]\n"
    "Summary: [Summary]\n\n"
    "The email is below:\n\n"
)

# Parameters that shold not be changed
SCOPES = ['https://www.googleapis.com/auth/gmail.readonly']
GMAIL_USER = 'me' # 'me' is a special value that indicates the authenticated user. Do not change this.
EMAIL_PASSWORD = os.getenv('EMAIL_PASSWORD')  # Use environment variables for security

if not EMAIL_PASSWORD:
    raise ValueError("EMAIL_PASSWORD environment variable not set.")

def authenticate_gmail():
    """Authenticate and create the Gmail API service."""
    creds = None
    if os.path.exists('token.json'):
        creds = Credentials.from_authorized_user_file('token.json', SCOPES)
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file('credentials.json', SCOPES)
            flow.redirect_uri = 'http://localhost:8080/'
            creds = flow.run_local_server(port=8080)
        with open('token.json', 'w') as token:
            token.write(creds.to_json())
    return build('gmail', 'v1', credentials=creds)

def get_messages(service):
    """Get a list of messages from the Gmail inbox."""
    results = service.users().messages().list(userId=GMAIL_USER, maxResults=INBOX_MAX_RESULTS).execute()
    messages = results.get('messages', [])
    return messages

def html_to_text(html_content):
    """Convert HTML content to plaintext."""
    soup = BeautifulSoup(html_content, 'html.parser')
    raw_text = soup.get_text()
    clean_text = re.sub(r'\s+', ' ', raw_text).strip()
    return clean_text

def get_message_details(service, message_id):
    """Get the details of a specific message, including labels."""
    msg = service.users().messages().get(userId=GMAIL_USER, id=message_id).execute()
    payload = msg['payload']
    headers = payload['headers']
    labels = msg.get('labelIds', [])
    
    # Extract subject, sender, and recipient from headers
    subject, sender, recipient = "", "", ""
    for header in headers:
        if header['name'] == 'Subject':
            subject = header['value']
        if header['name'] == 'From':
            sender = header['value']
        if header['name'] == 'To':
            recipient = header['value']
    
    # Extract email body; handle both plain text and HTML parts
    body = ""
    if 'parts' in payload:
        for part in payload['parts']:
            if "data" in part['body']:
                part_body = base64.urlsafe_b64decode(part['body']['data']).decode('utf-8')
                if part['mimeType'] == 'text/plain':
                    body += part_body
                elif part['mimeType'] == 'text/html':
                    body += html_to_text(part_body)
    else:
        body = base64.urlsafe_b64decode(payload['body']['data']).decode('utf-8')

    return subject, sender, recipient, body, labels

@retry(stop=stop_after_attempt(5), wait=wait_exponential(multiplier=1, min=4, max=10))
def summarize_text(text, subject, sender, recipient, labels, custom_instructions=CUSTOM_INSTRUCTIONS):
    """Summarize the given text using the Ollama API with additional context."""
    prompt = (
        f"{custom_instructions}\n\n"
        f"Subject: {subject}\n"
        f"From: {sender}\n"
        f"To: {recipient}\n"
        f"Labels: {labels}\n"
        f"Email Body:\n{text}\n\n"
        f"Analysis reminder: Don't forget about instructions 1-8.\n\n"
    )
    payload = {
        "model": MODEL,
        "prompt": prompt,
        "stream": False
    }
    response = requests.post(API_URL, json=payload)
    response.raise_for_status()
    
    json_response = response.json()
    summary = json_response.get('response', '')

    return summary

def send_email(subject, body, to_email=TO_EMAIL):
    """Send an email using smtplib."""
    msg = MIMEMultipart()
    msg['From'] = FROM_EMAIL
    msg['To'] = to_email
    msg['Subject'] = subject

    msg.attach(MIMEText(body, 'plain'))

    server = smtplib.SMTP('smtp.gmail.com', 587)
    server.starttls()
    server.login(FROM_EMAIL, EMAIL_PASSWORD)
    text = msg.as_string()
    server.sendmail(FROM_EMAIL, to_email, text)
    server.quit()

def main():
    """Main function to retrieve, summarize, and send email summaries."""
    service = authenticate_gmail()  # Authenticate Gmail API
    messages = get_messages(service)  # Fetch messages

    summaries = []  # List to store all summaries

    # Process each message and summarize
    for message in messages:
        msg_id = message['id']
        subject, sender, recipient, body, labels = get_message_details(service, msg_id)
        summary = summarize_text(body, subject, sender, recipient, labels)
        summaries.append(summary)
    
    # Consolidate all summaries into a single email body
    consolidated_summary = "\n\n".join(summaries)
    send_email("Consolidated Email Summaries", consolidated_summary)

if __name__ == '__main__':
    main()
```

### Conclusion

This Python script exemplifies the seamless integration of Ollama's self-hosted AI system with Gmail for efficient email summarization. While currently tailored for summarization, this foundational script holds vast potential for future enhancements, including dynamic email categorization and management. 

In today's rapidly evolving technological landscape, it's crucial to ask yourself daily how AI can enhance your life. Embrace this change and integrate AI into your routine to stay ahead of the curve. You GOT this!

I'd love to hear your thoughts and experiences with this script. Your comments and feedback are valuable to me!