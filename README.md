# Gmail Explorer Reader

###### By [nickesc](https://github.com/nickesc) / [N. Escobar](https://nickesc.com)

### *Project Idea:*

*Use the [`Gmail API`](https://developers.google.com/gmail/api/guides) to grab data about every email I've ever recieved over several email addresses, sift through it, and analyze it, visualizing different trends, patterns or interesting points.*

> This notebook can be run on its own to get the messages from your inbox. With the links at the top of the [`nbviewer` page](https://nbviewer.org/github/nickesc/Gmail-Explorer-Reader/blob/main/Gmail%20Explorer.ipynb), either [download the `.ipynb`](https://raw.githubusercontent.com/nickesc/Gmail-Explorer-Reader/main/Gmail%20Explorer.ipynb) and run it locally or [open the notebook in Binder](https://mybinder.org/v2/gh/nickesc/Gmail-Explorer-Reader/main?filepath=Gmail%20Explorer.ipynb) to run it online.

### Setting up the environment

##### Installing the Google client library (from the [`Gmail API Docs`](https://developers.google.com/gmail/api/guides)):


```python
!pip install --upgrade --upgrade-strategy=only-if-needed google-api-python-client google-auth-httplib2 google-auth-oauthlib
```

##### Installing ipywidgets, scikit-learn and seaborn:


```python
!pip install --upgrade --upgrade-strategy=only-if-needed ipywidgets scikit-learn seaborn
```

##### Importing all our packages:


```python
from __future__ import print_function

import os
import csv
import base64
import pandas as pd
import seaborn as sns
import numpy as np
import sklearn
import matplotlib.pyplot as plt


from IPython.display import clear_output, display
from ipywidgets import *
from tkinter import Tk, filedialog
from math import floor


# GMAIL API
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError
```

### Notebook options


```python
credentials = "credentials.json"
tokenName = "token.json"
dev=[False]
chunks=[10]

devToggle = widgets.ToggleButtons(
    options=['on', 'off'],
    description='Dev mode:',
    disabled=False,
    button_style='',
    value='off'
)
chunkSlider = widgets.IntSlider(
    value=10,
    min=1,
    max=20,
    step=1,
    description='Chunks:',
    orientation='horizontal',
    readout=True,
)
def devClick(change):
    dev[0]= not dev[0]
def chunkClick(change):
    chunks[0]=chunkSlider.value
    
devToggle.observe(devClick, names='value')
chunkSlider.observe(chunkClick, names='value')

display(chunkSlider,devToggle,widgets.Label("(dev mode may break code, only use if running the notebook locally)"))
```

### Reading Inboxes

The [`Gmail API`](https://developers.google.com/gmail/api/guides) can be used to return a paginated list of all messages in a user's inbox, however it only returns the message IDs. So, first we need to get all the IDs, then we need to get all the messages with the IDs

#### Accessing the account

First, the user needs to log in to their Google account and give it access to read everything. The  [`Gmail API Docs`](https://developers.google.com/gmail/api/guides) list these ase requirements for using the  [`Gmail API`](https://developers.google.com/gmail/api/guides):

> 1. [A Google Cloud Platform project with the API enabled.](https://developers.google.com/workspace/guides/create-project)
> 2. [Authorization credentials for a desktop application](https://developers.google.com/workspace/guides/create-credentials)
> 3. A Google account with Gmail enabled
>
> I found their resources for setting up the project unhelpful, and found [this Medium Article](https://towardsdatascience.com/extracting-metadata-from-medium-daily-digest-newsletters-via-gmail-api-97eee890a439) by [Sejal Dua](https://sejaldua.medium.com/) much more helpful.

The two files we need to communicate with the [`Gmail API`](https://developers.google.com/gmail/api/guides) are `credentials.json` and `token.json`. We get `credentials.json` during our setup from Google Cloud Console, but `token.json` is generated for the current session when we log in. These files are located in the `root` of the notebook, and the notebook assumes it has `credentials.json` unless you upload something.

> To load credentials or clear the currrent session token (to use a different account), use these snippets:

##### Load credentials:


```python
def uploadFile(change):
    clear_output()
    with open(credentials, "w+b") as i:
        i.write(upload.data[0])
    print("New",credentials,"uploaded.")
    
    
upload = FileUpload(accepts = '.json', multiple=False)
upload.observe(uploadFile, names='value')

print("Upload the credentials.json file from Google Cloud Console")
display(upload)
```

##### Clear session token:


```python
def clearToken(b):
    clear_output()
    if os.path.exists(tokenName):
        os.remove(tokenName)
    print("Session token cleared. Please run the next cell to log in.")
    display(clear)


clear = Button(description="Clear session token")
clear.on_click(clearToken)

print("Click to clear the current session token.")
display(clear)
```

##### Signing in

Running the next cell will take you to another tab to log in to your Gmail account. It can only be one of the accounts you've added as a test user, so be sure to select the right one. You will be asked to sign in and verify you want to use the app, and then it will give you an authorization code to plug back into the notebook. In `dev mode`, the authorization code isn't used and it authorizes automatically after you approve the app -- this cannot be used online because it relies on a localhost connection. 


```python
# If modifying these scopes, delete the file token.json.

SCOPES = ['https://www.googleapis.com/auth/gmail.readonly']
creds = None
# The file token.json stores the user's access and refresh tokens, and is
# created automatically when the authorization flow completes for the first
# time.
if os.path.exists(tokenName):
    creds = Credentials.from_authorized_user_file(tokenName, SCOPES)
# If there are no (valid) credentials available, let the user log in.
if not creds or not creds.valid:
    if creds and creds.expired and creds.refresh_token:
        creds.refresh(Request())
    else:
        flow = InstalledAppFlow.from_client_secrets_file(credentials, SCOPES)
        if dev[0]:
            creds = flow.run_local_server(port=0)
        else:
            creds = flow.run_console(authorization_code_message='Enter the authorization code and press enter: ')

        #creds = flow.run_local_server(port=0)
    # Save the credentials for the next run
    with open(tokenName, 'w') as token:
        token.write(creds.to_json())

try:
    service = build('gmail', 'v1', credentials=creds)
    print(
        "\nSigned in to",
        service.users().getProfile(userId='me').execute()["emailAddress"] +
        "; continue to the next cell.\n")
except:
    print(
        "\nService error, please retry or throw your hands up in confusion\n")
```

#### Getting message `id`s

To avoid enormous repsonses, the [`Gmail API`](https://developers.google.com/gmail/api/guides) returns paginated responses and lists of `id`s instead of full emails. So the program needs to loop through reponses to get all the `id`s. This actually allowed for a progress update to show how long it was taking (up to a few minutes for larger accounts) when it otherwise didn't look like it was working. It keeps asking for pages until it can't find a `nextPageToken`.

> This percent system divides the current number of IDs recieved by the total expected number listed in the user's profile data. It isn't a perfect solution, because deleted emails are collected but not counted towards the profile number, but it works fine to get across how quickly the API is working.
>
> This was initially written for tracking recieving messages, but it worked so well there that I just dropped the code here with a few tweaks. That code later had to be reworked (because it did not, in fact, work so well), but the original version is still here, because this percentage is just vanity, where as the other is functional as well. 


```python
def getPage(writer, percents, expected, current, pageToken=None):

    try:
        # Call the Gmail API

        results = service.users().messages().list(
            userId='me',
            pageToken=pageToken,
            maxResults=500,
            includeSpamTrash=True).execute()
        for pair in results["messages"]:
            percent = round((current / expected) * 100)
            if percent not in percents:
                percents.append(percent)
                if percent % 5 == 0:
                    print("%" + str(percent) + " complete...")
            writer.writerow([pair["id"]])
            current += 1
        if results["nextPageToken"]:
            getPage(writer, percents, expected, current,
                    results["nextPageToken"])
        else:
            print(results)

    except Exception as e:
        # TODO(developer) - Handle errors from gmail API.
        print(f'Messages end: no {e}')


def cleanCSV(fileName):
    clean = open(fileName, "w")
    writer = csv.writer(clean)
    writer.writerow(["id"])
    clean.close()
    print(fileName + " cleaned.\n")


def displayHead(fileName):
    data = pd.read_csv(fileName)
    display(data.head())
    print("...and " + str(data.shape[0] - 5) + " more rows")


def getIDs(name):

    profile = service.users().getProfile(userId='me').execute()

    percents = []
    expected = profile["messagesTotal"]
    current = 0

    fileName = name + "IDs.csv"
    cleanCSV(fileName)
    print("Finding ~" + str(profile["messagesTotal"]) + " messages...")

    IDFile = open(fileName, 'a')
    writer = csv.writer(IDFile)
    getPage(writer, percents, expected, current)
    IDFile.close()

    messageCount = len(pd.read_csv(fileName))

    print("Finshed collecting " + str(messageCount) + " out of " +
          str(profile["messagesTotal"]) + " messages from " +
          profile["emailAddress"])
    displayHead(fileName)
    return fileName
```

The specific call we make to the [`Gmail API`](https://developers.google.com/gmail/api/guides) --

```python
results = service.users().messages().list(userId = 'me', pageToken = pageToken, maxResults = 500, 
                                          includeSpamTrash = True).execute()
```

-- grabs a list of the next 500 results, instead of the default 100 (`maxResults = 500`), from the user's inbox, including pulling from usually ignored Spam and Trash folders (`includeSpamTrash = True`).

This returns:

```python
{'messages': [
        {
            'id': '...', 
            'threadId': '...'
        }, 
        {
            'id': '...', 
            'threadId': '...'
        }...
    ]
}
```

Each entry in messages is a different message, with an `id` and a `threadId`. The `id` and `threadId` are usually the same, but when you respond to an email it starts a thread, and the entire thread shares the `threadId` to associate them, while the individual message keeps a unique `id`. Here, all we care about is the `id`, so we add each `id` into a `.csv` and dump the rest of the information.

#### Selecting an account and running the program


```python
accounts = ["jg", "gd", "ne", "nm"]
fields = [
    'id', 'received', 'delivered-to', 'to', 'from', 'subject', 'labels',
    'sizeEstimate', 'threadId', 'internalDate', 'body'
]

account = widgets.RadioButtons(options=accounts,
                               value=accounts[0],
                               description='Account')
display(account)
```

Finally, the program asks the user to choose which account identifiers to use for the ID file, in case you have multiples accounts and want to make multiple ID files. Account identifiers are based of my own accounts, but have no impact on anything but file name. Run the next cell to start collecting IDs using the above functions.


```python
fileName = getIDs(account.value)
```

#### Converting `id`s to messages

Now that we have a list of `id`s, we need to to turn them into messages to tell anything usefull from them. The first part of that is prepping our files, including creating the message output `messages.csv` where all out final data will go, clearing old data, adding headers, and setting up readers and writers for our output `.csv`s.

>If `messages.csv` needs to be prepped to do ___the first or a fresh take___, click this button for a clean file with a header:

##### Clean `messages.csv`


```python
messageFileName = "messages.csv"


def cleanMessages(b):
    clear_output()
    messageFile = open(messageFileName, 'w')
    messageWriter = csv.DictWriter(messageFile, fieldnames=fields)
    messageWriter.writeheader()
    messageFile.close()
    print(messageFileName, "cleaned.\n")
    display(clean)


clean = Button(description="Clean " + messageFileName)
clean.on_click(cleanMessages)

display(clean)
```


```python
IDFile = open(fileName, 'r')
IDReader = csv.reader(IDFile, delimiter=',')
ids = []

messageFile = open(messageFileName, 'a')
messageWriter = csv.DictWriter(messageFile, fieldnames=fields)

profile = service.users().getProfile(userId='me').execute()

currPercent = 0

for row in IDReader:
    ids.append(row[0])
ids.pop(0)

#chunks = 10


def splitList(data, chunks):
    length = len(data)
    n = floor(length / chunks)

    for i in range(0, len(data), n):
        yield data[i:i + n]


ids = list(splitList(ids, chunks[0]))
idIndex = 0
if len(ids) > chunks[0]:
    while len(ids[-1]) != 0:
        ids[idIndex].append(ids[-1][0])
        ids[-1].pop(0)
        if idIndex == chunks[0] - 1:
            idIndex = 0
        else:
            idIndex += 1
    ids.pop(-1)

print('Files ready. Messages will request in ' + str(len(ids)) +
      ' chunks of ~' + str(len(ids[0])))
```

#### Ten-percent of your inbox

So we're ready to to ask for all our emails with our big list all at once right? Nope! Unfortunately , with larger inboxes, the notebook times out before it can request all the emails. The solution I found was to split the list of IDs into ten evenly-divided parts, and make ten percent of the calls for the inbox in a cell at a time. It makes calls until one-hundred-percent of the messages in the `.csv` are requested.

> In testing, this allowed enough time for all emails to be requested and received on an inbox with ~37,000 emails. It's best to just run all 10 cells at once and walk away -- come back after a good night's sleep and hope nothing timed out!
>
> If you do run into problems, the number of chunks the program splits the inbox into can be adjusted in the options near the top of the notebook. Increasing the number of chunks will decrease the number of messages requested in each cell, but will also require you to run the `tenPercent()` function again. This can be done either by adding cells or rerunning the current ones.



Each message request is made with this call --

``` python
message = service.users().messages().get(userId = 'me', id = next(IDReader)[0], format = 'full').execute()
```

-- which asks the [`Gmail API`](https://developers.google.com/gmail/api/guides) to send the full information (`format = 'full'`) for the next `id` in the list (`id = next(IDReader)[0]`).


```python
def convertLabels(labels):
    string=""
    x=0
    for item in labels:
        if x==0:
            string=item
            x+=1
        else:
            string=string+","+str(item)
    return string


def getMessage(message):

    messageResult = {}
    try:
        content = message['payload']['body']['data']
        #msg_body = base64.urlsafe_b64decode(content).decode('utf-8')
    except:
        content = ''
        for part in message['payload']['parts']:
            try:
                #msg_part = base64.urlsafe_b64decode(part['body']['data']).decode('utf-8')
                msg_part = part['body']['data']
            except:
                msg_part = ""
            finally:
                content = content + msg_part
    msg_body = content
    #msg_body = base64.urlsafe_b64decode(content).decode('utf-8')

    messageResult['threadId'] = message['threadId']
    messageResult['id'] = message['id']

    rec = False

    for header in message['payload']['headers']:
        if header['name'] == 'Delivered-To':
            messageResult['delivered-to'] = header['value']
        if header['name'] == 'To':
            messageResult['to'] = header['value']
        if header['name'] == 'From':
            messageResult['from'] = header['value']
        if header['name'] == 'Received' and rec == False:
            messageResult['received'] = header['value']
            rec = True
        if header['name'] == 'Subject':
            messageResult['subject'] = header['value']

    messageResult['labels'] = convertLabels(message['labelIds'])
    messageResult['sizeEstimate'] = message['sizeEstimate']
    messageResult['internalDate'] = message['internalDate']
    messageResult['body'] = msg_body

    return messageResult


def tenPercent(ids, percentIndex):

    print("%" + str(floor((percentIndex / len(ids)) * 100)) + " complete...")

    currIds = ids[percentIndex]

    messageRepo = []

    for ID in currIds:
        try:
            message = service.users().messages().get(userId='me',
                                                     id=ID,
                                                     format='full').execute()
            messageResult = getMessage(message)
            #print(message)
            messageRepo.append(messageResult)
        except Exception as e: print(e)

    messageWriter.writerows(messageRepo)

    percentIndex += 1
    print("%" + str(floor((percentIndex / len(ids)) * 100)) + " complete...")

    return (percentIndex)


print(
    "Ready to start requesting messages. Run the next few cells one-after-another.\n"
    + "This will take a long time (hours) for inboxes with a lot of messages.")
```


```python
currPercent = tenPercent(ids,currPercent)
```


```python
currPercent = tenPercent(ids,currPercent)
```


```python
currPercent = tenPercent(ids,currPercent)
```


```python
currPercent = tenPercent(ids,currPercent)
```


```python
currPercent = tenPercent(ids,currPercent)
```


```python
currPercent = tenPercent(ids,currPercent)
```


```python
currPercent = tenPercent(ids,currPercent)
```


```python
currPercent = tenPercent(ids,currPercent)
```


```python
currPercent = tenPercent(ids,currPercent)
```


```python
currPercent = tenPercent(ids,currPercent)
```

#### Cleaning the data up

This produces a lot of data we don't want, unfortunately. In addition to collecting the information, we make it a little easier to work with by parsing out some of the information ahead of time. The data comes in this form:

```python
{
    "internalDate": "...", 
    "historyId": "...",
    "payload": { 
        "body": { 
            "data": "...", 
            "attachmentId": "...", 
            "size": ..., 
        },
        "mimeType": "...", 
        "partId": "...", 
        "filename": "...", 
        "headers": [ 
            {
                "name": "...",
                "value": "...",
            }...
        ],
        "parts": [...]...
    },
    "snippet": "...", 
    "sizeEstimate": ..., 
    "threadId": "...",
    "labelIds": ["..."...],
    "id": "...",
}
```

Email addresses are hidden in header tags and the content of the email is hidden away in data tags. Thankfully the data is encoded, so it uses less space, but it means that we'll need to [decode](https://docs.python.org/3/library/base64.html) it on the other end instead of here.

I've turned this data into just what is useful to us. For each message we get:

```python
{
    "received": "...",
    "delivered-to": "...",
    "to": "...",
    "from": "...",
    "subject": "...",
    "labels": ["..."...],
    "sizeEstimate": ...,
    "threadId": "...",
    "internalDate": ..., 
    "body": ...
}
```

The data outputs to `messages.csv` in the notebook's `root` directory.


```python
IDFile.close()
messageFile.close()

inboxes = pd.read_csv(messageFileName)

messageCount = len(inboxes)

print("Retrieved " + str(messageCount) + " out of ~" +
      str(profile["messagesTotal"]) + " expected messages")

inboxes.head()
```

#### Downloading the data

If you run this locally, you should be able to just find the `messages.csv` file in your filesystem. If you're running this on Binder, however, you'll need a way to download the created `.csv`. Thankfully, there's a way to download a `pandas` dataframe into a `.csv`, which means we just need to output a link.


```python
from IPython.display import HTML

def create_download_link( df, title = "Download your messages", filename = "data.csv"):  
    csv = df.to_csv()
    b64 = base64.b64encode(csv.encode())
    payload = b64.decode()
    html = '<a download="{filename}" href="data:text/csv;base64,{payload}" target="_blank">{title}</a>'
    html = html.format(payload=payload,title=title,filename=filename)
    return HTML(html)

create_download_link(inboxes)
```

### References

[Extracting Metadata from Medium Daily Digest Newsletters via Gmail API](https://towardsdatascience.com/extracting-metadata-from-medium-daily-digest-newsletters-via-gmail-api-97eee890a439) by [Sejal Dua](https://sejaldua.medium.com/)

[`Gmail API Dodumentation`](https://developers.google.com/gmail/api/guides)

[`Gmail API Python Dodumentation`](https://developers.google.com/resources/api-libraries/documentation/gmail/v1/python/latest/index.html)

[`ipywidgets Dodumentation`](https://ipywidgets.readthedocs.io/en/stable/)

[`tkinter Dodumentation`](https://docs.python.org/3/library/tk.html)
