# 🏀 NBA Game Day Notification System

This project is an **AWS Lambda-powered notification system** that sends NBA game updates via **Amazon SNS (Simple Notification Service)**. It fetches game details from **SportsData.io** and notifies subscribed users via email. The function runs on a scheduled **Amazon EventBridge** rule.

##  Features

- Fetches NBA game data from **SportsData.io API**
- Sends real-time updates on **game status, scores, and channels**
- Uses **AWS Lambda, SNS, and EventBridge** for automation
- Supports multiple game statuses: **Final, In Progress, and Scheduled**

---

##  Prerequisites

Before deploying, ensure you have:

- **AWS Account** with Lambda, SNS, and EventBridge permissions
- **SportsData.io API Key** (Create a free API key [here](https://sportsdata.io))
- **Email Subscription** to the SNS topic

---

##  Setup Instructions


### 1️⃣ Create an SNS Topic

- Open AWS SNS and create a new **topic** named `gd_topic`.
- Subscribe an **email address** to this topic and confirm the email subscription.


![sns topic](https://github.com/Joseph-Ibeh/NBA-Gameday-Notifier/blob/main/Assets/create%20topic.png)

![subscribe sns topic](https://github.com/Joseph-Ibeh/NBA-Gameday-Notifier/blob/main/Assets/subscribe%20to%20topic.png)


### 2️⃣ Create an IAM Role for Lambda

- Open **IAM** → **Roles** → **Create Role**.
- Select **AWS Service → Lambda**.
- Attach the **AmazonSNSFullAccess** policy.
- Create a custom **JSON policy** with the following:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:us-east-1:XXXXXXXXXXXX:gd_topic"
    }
  ]
}
```


Replace `XXXXXXXXXXXX` with your AWS Account ID.

- Name the role **gd_lambda_role** and attach it to your Lambda function.

![role and policy](https://github.com/Joseph-Ibeh/NBA-Gameday-Notifier/blob/main/Assets/gd%20role%20and%20policy.png)


### 3️⃣ Deploy the AWS Lambda Function

- Navigate to **AWS Lambda** → **Create Function** → **Author from Scratch**.
- Function Name: **gd_notification**
- Runtime: **Python 3.13**
- Attach **gd_lambda_role** to the function.
- Paste the following **Python script** into the function editor:
  

```python
import os
import json
import urllib.request
import boto3
from datetime import datetime, timedelta, timezone

# Function to format NBA game data
def format_game_data(game):
    status = game.get("Status", "Unknown")
    away_team = game.get("AwayTeam", "Unknown")
    home_team = game.get("HomeTeam", "Unknown")
    final_score = f"{game.get('AwayTeamScore', 'N/A')}-{game.get('HomeTeamScore', 'N/A')}"
    start_time = game.get("DateTime", "Unknown")
    channel = game.get("Channel", "Unknown")

    if status == "Final":
        return f"Game Status: {status}\n{away_team} vs {home_team}\nFinal Score: {final_score}\nStart Time: {start_time}\nChannel: {channel}\n"
    elif status == "InProgress":
        return f"Game Status: {status}\n{away_team} vs {home_team}\nCurrent Score: {final_score}\nChannel: {channel}\n"
    elif status == "Scheduled":
        return f"Game Status: {status}\n{away_team} vs {home_team}\nStart Time: {start_time}\nChannel: {channel}\n"
    else:
        return f"Game Status: {status}\n{away_team} vs {home_team}\nDetails are unavailable.\n"

# Lambda function handler
def lambda_handler(event, context):
    api_key = os.getenv("NBA_API_KEY")
    sns_topic_arn = os.getenv("SNS_TOPIC_ARN")
    sns_client = boto3.client("sns")

    utc_now = datetime.now(timezone.utc)
    today_date = (utc_now - timedelta(hours=6)).strftime("%Y-%m-%d")
    api_url = f"https://api.sportsdata.io/v3/nba/scores/json/GamesByDate/{today_date}?key={api_key}"

    try:
        with urllib.request.urlopen(api_url) as response:
            data = json.loads(response.read().decode())
    except Exception as e:
        print(f"Error fetching data: {e}")
        return {"statusCode": 500, "body": "Error fetching data"}

    messages = [format_game_data(game) for game in data]
    final_message = "\n---\n".join(messages) if messages else "No games available for today."

    try:
        sns_client.publish(TopicArn=sns_topic_arn, Message=final_message, Subject="NBA Game Updates")
        print("Message published to SNS successfully.")
    except Exception as e:
        print(f"Error publishing to SNS: {e}")
        return {"statusCode": 500, "body": "Error publishing to SNS"}

    return {"statusCode": 200, "body": "Data processed and sent to SNS"}
```

![py script](https://github.com/Joseph-Ibeh/NBA-Gameday-Notifier/blob/main/Assets/python%20script.png)

- Click **Deploy**.

![test](https://github.com/Joseph-Ibeh/NBA-Gameday-Notifier/blob/main/Assets/test%20status%20.png)

- Add **Environment Variables**:

  - `NBA_API_KEY`: Your SportsData.io API Key
  - `SNS_TOPIC_ARN`: ARN of your SNS Topic (e.g., `arn:aws:sns:us-east-1:XXXXXXXXXXXX:gd_topic`)
 
![Env variable](https://github.com/Joseph-Ibeh/NBA-Gameday-Notifier/blob/main/Assets/Env%20variable%20setup.png)


- Increase **timeout** under _General Configuration_ to **5 seconds or more**.

![timeout](https://github.com/Joseph-Ibeh/NBA-Gameday-Notifier/blob/main/Assets/increase%20time%20out.png)


### 4️⃣ Test the Lambda Function

- Click **Test**, create a test event, and run it.
- Check your **email inbox** for an NBA game update notification!

![email update](https://github.com/Joseph-Ibeh/NBA-Gameday-Notifier/blob/main/Assets/email%20score%20update.png)

![email update 2](https://github.com/Joseph-Ibeh/NBA-Gameday-Notifier/blob/main/Assets/email%20score%20update%202.png)


### 5️⃣ Automate with AWS EventBridge

- Open **Amazon EventBridge** → **Create Rule**
- Rule Name: **gd_rule**
- **Schedule Pattern**:
  ```
  0 9-23/2,0-2/2 * * ? *
  ```
  _(Runs every 2 hours from 9 AM - 11 PM UTC and 12 AM - 2 AM UTC)_
- **Target**: AWS Lambda → **NBA-GameDay-Notifier**
- Click **Create Rule**.

![Event bridge](https://github.com/Joseph-Ibeh/NBA-Gameday-Notifier/blob/main/Assets/event%20bridge%20schedule.png)

---


## Email Notification


![email update](https://github.com/Joseph-Ibeh/NBA-Gameday-Notifier/blob/main/Assets/email%20score%20update.png)

![email update 2](https://github.com/Joseph-Ibeh/NBA-Gameday-Notifier/blob/main/Assets/email%20score%20update%202.png)
---

## Summary

- **AWS SNS** sends game notifications to subscribed emails.
- **AWS Lambda** fetches NBA data and publishes updates.
- **Amazon EventBridge** schedules function execution.


## 🛠 Challenges Encountered  
### Lambda Execution Timeout  
Initially, the function hit the default execution timeout limit. To resolve this, I:  
- Increased the Lambda timeout to **5 seconds**.  

