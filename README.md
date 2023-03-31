---
jupyter:
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
  language_info:
    codemirror_mode:
      name: ipython
      version: 3
    file_extension: .py
    mimetype: text/x-python
    name: python
    nbconvert_exporter: python
    pygments_lexer: ipython3
    version: 3.11.0rc2
  nbformat: 4
  nbformat_minor: 5
---

::: {#d773fd5e .cell .markdown}
# 1. Telegram Channel Scrapper {#1-telegram-channel-scrapper}
:::

::: {#7ebbbb5c .cell .markdown}
Автор задачи: Lantern

Lantern's Discover feature scraps content from a lot of sources so it'll
always be available for usage uncensored and to have a backup in case of
a takedown.

Make a Telegram channel scraper that either saves the content locally or
returns a JSON blob that another service can consume to download the
content.

The content in this case are the videos and images for that channel.
:::

::: {#13e7be9f .cell .markdown}
# Intent: write a linear code for storing the messafes and media of channel in a tar archive and upload this to the AWS S3 bucket, reflect the new archive in a index.html of s3 bucket {#intent-write-a-linear-code-for-storing-the-messafes-and-media-of-channel-in-a-tar-archive-and-upload-this-to-the-aws-s3-bucket-reflect-the-new-archive-in-a-indexhtml-of-s3-bucket}
:::

::: {#db6ad269 .cell .code execution_count="1"}
``` python
#!python -m pip install --upgrade telethon
```
:::

::: {#ba053b3f .cell .code execution_count="2"}
``` python
#!pip install xtarfile boto3
```
:::

::: {#be0f5364 .cell .code execution_count="3"}
``` python
import pandas as pd , os, json
```
:::

::: {#41d3cb81 .cell .code execution_count="4"}
``` python
from telethon import TelegramClient
```
:::

::: {#ac110474 .cell .markdown}
# Get Telegram creds from json
:::

::: {#5b6d36e0 .cell .code execution_count="5"}
``` python
with open('creds.json') as json_file:
    creds = json.load(json_file)
    
```
:::

::: {#1ef25fd0 .cell .code execution_count="6"}
``` python
api_id = creds["api_id"]
api_hash = creds['api_hash']
phone = creds['phone']
username = creds["username"]
```
:::

::: {#df4128ac .cell .code execution_count="7"}
``` python
# One of telegram chanels for  scraping
```
:::

::: {#595c094c .cell .code execution_count="8"}
``` python
chat = "vatnoeboloto"
chat = "YouTubot"
chat = "Kushnar_media"
chat = "insider_uke"
```
:::

::: {#18612ced .cell .markdown}
# Preparing working environment
:::

::: {#69e83a54 .cell .code execution_count="9"}
``` python
chat_addr = "https://t.me/{}".format(chat)
print(chat_addr)
download_folder = "download_of_{}".format(chat)
isExist = os.path.exists(download_folder)
if not isExist:
   os.makedirs(download_folder)
```

::: {.output .stream .stdout}
    https://t.me/insider_uke
:::
:::

::: {#c2af44a3 .cell .markdown}
# scraping messages and media. media files are writing in local folder {#scraping-messages-and-media-media-files-are-writing-in-local-folder}
:::

::: {#5b0e645f .cell .code execution_count="24"}
``` python
data = [] 
n = 0
n_max = 30
async with TelegramClient(username, api_id, api_hash) as client:
    async for message in client.iter_messages(chat_addr, reverse=False):
        n = n + 1
        if (n > n_max):
            break
        data.append([message, message.sender_id, message.text, message.date, message.id, message.post_author, message.views, message.peer_id.channel_id , message.photo ])
        if (message.photo or message.video) :
            await message.download_media("{}\msg-{}-{}".format(download_folder,message.peer_id.channel_id,message.id))
```
:::

::: {#1c61850f .cell .markdown}
# creating dataframe for observing and postproduction
:::

::: {#10761cfe .cell .code execution_count="23"}
``` python
columns=["message","message.sender_id", "message.text"," message.date", "message.id",  "message.post_author", "message.views", "message.peer_id.channel_id", "message.photo" ]
df = pd.DataFrame(data, columns=columns) # creates a new dataframe
#df['message.text.removed_emojis'] = df.apply(lambda row : remove_emojis(row["message.text"]), axis = 1)
df = df.reset_index()
```
:::

::: {#1c6d83f0 .cell .markdown}
# saving messages In json file
:::

::: {#3fd9e243 .cell .code execution_count="12" scrolled="true"}
``` python
filename = "messages_of_channel-{}".format(chat)
df.to_csv('{}.csv'.format(filename), encoding='utf-8')
df.drop(columns=['message',"message.photo"]).to_json('{}.json'.format(filename))
```
:::

::: {#1e72ce76 .cell .code execution_count="13"}
``` python
from os import walk
media_list = []
for (dirpath, dirnames, filenames) in walk(download_folder):
    media_list.extend(filenames)
    break
#f    
```
:::

::: {#19018999 .cell .markdown}
# creating tarfile with messages (csv, json) and subfolder with media
:::

::: {#276b4585 .cell .code execution_count="14"}
``` python
import xtarfile as tarfile
with tarfile.open('{}.tar'.format(filename), 'w') as archive:
    archive.add('{}.csv'.format(filename))
    archive.add('{}.json'.format(filename))
    for i in media_list:
        archive.add('{}/{}'.format(download_folder,i))
```
:::

::: {#8002abe0 .cell .markdown}
# uploading tar to AWS s3 bucket with public access
:::

::: {#0c0aee78 .cell .code execution_count="15"}
``` python
import boto3
s3 = boto3.resource("s3")
```
:::

::: {#221f0c8e .cell .code execution_count="16"}
``` python
s3.meta.client.upload_file(
    Filename='{}.tar'.format(filename),
    Bucket="internet-without-borders",
    Key='{}.tar'.format(filename),
)
```
:::

::: {#599165a5 .cell .markdown}
# updating index.html in s3 bucket with public access {#updating-indexhtml-in-s3-bucket-with-public-access}
:::

::: {#879f2c25 .cell .code execution_count="17"}
``` python
my_bucket = s3.Bucket('internet-without-borders')
s3_tars_list = []
for my_bucket_object in my_bucket.objects.all():
    if ".tar" in my_bucket_object.key: 
        s3_tars_list.append("<br><a href={}>{}</a>".format(my_bucket_object.key, my_bucket_object.key))
s3_tars_list
```

::: {.output .execute_result execution_count="17"}
    ['<br><a href=messages_of_channel-Kushnar_media.tar>messages_of_channel-Kushnar_media.tar</a>',
     '<br><a href=messages_of_channel-YouTubot.tar>messages_of_channel-YouTubot.tar</a>',
     '<br><a href=messages_of_channel-insider_uke.tar>messages_of_channel-insider_uke.tar</a>',
     '<br><a href=messages_of_channel-vatnoeboloto.tar>messages_of_channel-vatnoeboloto.tar</a>']
:::
:::

::: {#529eca77 .cell .code execution_count="18"}
``` python
index_text = "<html><body>{}</body></html>".format("".join(s3_tars_list))
index_text
```

::: {.output .execute_result execution_count="18"}
    '<html><body><br><a href=messages_of_channel-Kushnar_media.tar>messages_of_channel-Kushnar_media.tar</a><br><a href=messages_of_channel-YouTubot.tar>messages_of_channel-YouTubot.tar</a><br><a href=messages_of_channel-insider_uke.tar>messages_of_channel-insider_uke.tar</a><br><a href=messages_of_channel-vatnoeboloto.tar>messages_of_channel-vatnoeboloto.tar</a></body></html>'
:::
:::

::: {#39b90bb9 .cell .code execution_count="19"}
``` python
with open('index.html', 'w') as f:
    f.write(index_text)
```
:::

::: {#bcf00fb3 .cell .code execution_count="20"}
``` python
s3.meta.client.upload_file(
    Filename="index.html",
    Bucket="internet-without-borders",
    Key="index.html",
)
```
:::

::: {#c98acb94 .cell .markdown}
# The public URL of s3-based web-site with archives:

    https://internet-without-borders.s3.eu-central-1.amazonaws.com/index.html
:::

::: {#1e6655ec .cell .markdown}
# TODO:

## UTF-8 in messages.text {#utf-8-in-messagestext}

## media download with multithreading

## docker image

## deploy in AWS EKS or AWS Lambda

## public API
:::

::: {#6e63bd54 .cell .code}
``` python
```
:::
