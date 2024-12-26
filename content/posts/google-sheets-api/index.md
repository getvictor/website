+++
title = "How to quickly edit Google Sheets spreadsheet using the API"
description = "How to get a Google Sheets API key and edit a Google Sheets spreadsheet"
authors = ["Victor Lyuboslavsky"]
image = "google-sheets-headline.png"
date = 2024-12-18
categories = ["Software Development"]
tags = ["Google Sheets", "Google APIs", "Golang"]
draft = false
+++

- [Get a Google Sheets API key](#get-a-google-sheets-api-key)
- [Share a Google Sheets spreadsheet with the service account](#share-a-google-sheets-spreadsheet-with-the-service-account)
- [Edit a Google Sheets spreadsheet using the API](#edit-a-google-sheets-spreadsheet-using-the-api)

When you need to edit a Google Sheets spreadsheet quickly, you can use the Google Sheets API. The API allows you to
programmatically read, write, and update data in a Google Sheets spreadsheet. However, following the
[Google Sheets API documentation](https://developers.google.com/sheets/api/guides/concepts) can be overwhelming. In this
article, we will show you how to get a Google Sheets API key and edit a Google Sheets spreadsheet using the API.

## User authentication (OAuth) vs. API key (JWT)

The problem is that the Google API documentation focuses on the [OAuth 2.0](https://oauth.net/2/) user authentication
flow. This flow is useful when you need to access Google Sheets on behalf of a user. For example, you're creating your
web app, and you need to read or write data in a Google Sheets spreadsheet owned by your web app user. The OAuth
standard allows Google to authenticate the user and authorize your app to access the user's data without exposing the
user's credentials to your web app. The OAuth flow interacts with three parties -- the user, your web app, and Google.

In our case, we want to access a specific Google Sheets spreadsheet programmatically without user interaction. We can
use the API key (JWT) authentication method. JWT stands for JSON Web Token, a standard for securely transmitting
information between two parties. This method allows us to access Google Sheets programmatically without user
interaction. The API key (JWT) method interacts with two parties: your app and Google.

## Get a Google Sheets API key

To get a Google Sheets API key, follow these steps:

1. Go to the [Google Cloud Console](https://console.cloud.google.com/).
2. Create a new project or select an existing project.
   {{< figure src="google-cloud-new-project.png" alt="Welcome to Google Cloud screen, with a modal to select a project. The NEW PROJECT text in the top right of the modal is highlighted.">}}
3. In the new project, go to **APIs & Services** and enable the **Google Sheets API**. After enabling it, you should see
   it in the **Enabled APIs & services** list.
   {{< figure src="google-cloud-enabled-sheets-api.png" alt="Google Cloud APIs & Services console with the Google Sheets API enabled.">}}
4. Go to **APIs & Services** > **Credentials** and create a new Service Account. This account does not need any optional
   permissions.
   {{< figure src="google-cloud-service-account.png" alt="Google Cloud IAM & Admin screen with a service account.">}}
5. Create a new JSON key for the service account.
   {{< figure src="google-cloud-service-account-create-new-key.png" alt="Google Cloud service account detailed view. The KEYS tab is selected. ADD KEY button is pressed, showing the option to Create new key.">}}
6. After creating the key, the JSON file will be automatically downloaded to your computer. This file contains the
   credentials for your service account. Keep it secure. For example, we received
   `axial-paratext-444915-f9-8ef1de636587.json` with the following content:

```json
{
  "type": "service_account",
  "project_id": "axial-paratext-444915-f9",
  "private_key_id": "8ef1de636587238a028addaa8be9dbdf1d406420",
  "private_key": "-----BEGIN PRIVATE KEY-----\nMIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDGFXmEc6VK0TO9\n2E/LDel4gTYl1u8uZGtX16B7Lo4ufM7ics3h9Gyi1lJMcGrHruGEzatDeTRclILd\nLhLwrckfl3IF9MIsqwaEkHk7YnUXj9zGl+v8LTGJL0ycQ9hVdoD6cCOOAmghLj8F\n9Sl6KQ5PHGbBUUL4qi8uExKY4tQOrqol1Pi3RPpAOCR6BLC/ZFPp+4e4HRhF+DMD\nI1QX8QwPit9XdIomnZPUL5sGD+q4cp1gHLBuBp2ehyiFI5MGhgzvCIQzaTExw7GK\nqrjYMjBKXaFRqGpZJWJMdVmGGHpmZLCL/wQmhujlThrF0FO8BHAGriAvUgDbH7m/\nMzGRk/KzAgMBAAECggEAE7LwBkWP8xxR9nfMG6fzB3pmHaY93BG9gRtfCNEM77+W\nvXtoUSfDJACHZ7WoUNpp8BCaDxg/JlPYndFmrcvCnCMuAjygkNujRsytWcQFXAYB\nETjrjYUbD4cGKeYvXfRuiDldt9Iyc9ZLCzch3FW36BMtft0reVpHXeAksdKg/yKf\nhM4jw3tQMu5JR3trLHtqwaA8VUav7I2Qn8nxEbB/0+AUatqpDOp/hQNTN7MGZ/i7\n4V0538U7C9RYDzk9hBBT2/IegGixlL0lX2V6LjYlRcEgC0PLuKF7gM/RRnorNPnv\nHfHxyZt6/MyI8RLRwwv05ZSITaOj66lXxReVsMXxhQKBgQD+vJPrvKD7mGiXRQto\nh1LJqPcyknzLzxf2OX5vZyF8asdroU0sRy8pYYHy8JCPlkOJ0fj9kq6R79W+rmn3\npFkvwRY9dUcJLpoMAMEfO4wQp3QKpxdkjMS8xGcEVOIZacAHCof7uUwrHUUcqRIq\nwrgZcj5P8ZjwsLmuqLNeXqFYNwKBgQDHEPf4PCyjieF+aGvaOPLfn/lRBbEQGlrg\n9Z09UXpxcW39RMq7MkS+U9m88Kn9MsEK3umJdP+s5m8ddVdIVgZLj96Ufn52RzRT\ne8crSjCVC3oQaloScvOBSQA1Z3Bn+QstIko042i2qTNJWMArdCJe9uRbwL1hqEvx\n+LtNPniDZQKBgQDX4g9maFzx/G8fS+doNc8mkmi01kqnGyJOjNknJnrNi1zoTTIv\nBUDly/oqXk/VMF6ajXV7yPTjPyOhTwUFV6Yx/2yOtzZ1hKYO6BDDHF8Ouitw37zG\nfTo6VCSOGjXnnaSdEwK9hYMUwuCQcoSv8oe9IQHIFJMt4EfsypIAtyf7rwKBgFtC\ntzvRcnGC+6K1AoTnyMimkWkIn/UO8Azj7TM4UFcDtnX+/KY3VHahAFhzSKswgnmW\nWiBPSAufFN+/dMVP0tD/Yv5Ww2k8GYwQWe3JtF4QBeTSrPp6QpJJwlO5WToBXZNS\nfgyjGNVs2ntMucTyF/PLYkOCKBBGVJLZAh1Wf29VAoGADf7a1l8kDKgokm6pc4qG\nzb97GMk1CpE0dGl31dvx2ilckDVP354yfWEwVXWWVfVSq/LQdJVgkdArYbAPdsPb\nYuUfNwXMSp/OjmEL2QyC2zRm+2ZZZt5bcnPRbYETzb2An8kDYX49vwgBLJXpLOmt\nlCvxUDyoASHgAMu+OlqHIh4=\n-----END PRIVATE KEY-----\n",
  "client_email": "example@axial-paratext-444915-f9.iam.gserviceaccount.com",
  "client_id": "114906617333001451487",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/example%40axial-paratext-444915-f9.iam.gserviceaccount.com",
  "universe_domain": "googleapis.com"
}
```

## Share a Google Sheets spreadsheet with the service account

Share the spreadsheet with the service account email address to allow the account to access a Google Sheets spreadsheet.
In our case, the service account email is `example@axial-paratext-444915-f9.iam.gserviceaccount.com`.

{{< figure src="google-sheets-share-with-service-account.png" alt="Google Sheets spreadsheet with the Share modal open. The spreadsheet is shared with the service account with editor permissions.">}}

Note the spreadsheet ID from the URL. For example, in the URL
`https://docs.google.com/spreadsheets/d/1QCtnB6MXfFJLZsBE1E2vq5FxKBgh1Q0s727wRxFkmX4/edit`, the spreadsheet ID is
`1QCtnB6MXfFJLZsBE1E2vq5FxKBgh1Q0s727wRxFkmX4`.

## Edit a Google Sheets spreadsheet using the API

Now that we have the Google Sheets API key and editor permissions for the target spreadsheet, we can edit it using the
API. For this example, we will use the Go programming language.

In an empty directory, create a Go project and get the necessary dependencies:

```bash
go mod init google-sheets-api
go get golang.org/x/oauth2@v0.24.0
go get google.golang.org/api@v0.211.0
```

Copy the JSON key file to the project directory and rename it to `credentials.json`.

Create a new Go file, `main.go`, with the following content:

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    "golang.org/x/oauth2/google"
    "google.golang.org/api/option"
    "google.golang.org/api/sheets/v4"
)

const spreadsheetId = "1QCtnB6MXfFJLZsBE1E2vq5FxKBgh1Q0s727wRxFkmX4"

func main() {
    ctx := context.Background()

    serviceAccountKey, err := os.ReadFile("credentials.json")
    if err != nil {
       log.Fatalf("Unable to read client secret file: %v", err)
    }

    cfg, err := google.JWTConfigFromJSON(serviceAccountKey, sheets.SpreadsheetsScope)
    if err != nil {
       log.Fatalf("Unable to parse client secret file to config: %v", err)
    }
    client := cfg.Client(ctx)

    srv, err := sheets.NewService(ctx, option.WithHTTPClient(client))
    if err != nil {
       log.Fatalf("Unable to retrieve Sheets client: %v", err)
    }

    readRange := "Sheet1!A2:B2"
    resp, err := srv.Spreadsheets.Values.Get(spreadsheetId, readRange).Do()
    if err != nil {
       log.Fatalf("Unable to retrieve data from sheet: %v", err)
    }

    if len(resp.Values) == 0 {
       fmt.Println("No data found.")
    } else {
       fmt.Println("Date, Value:")
       for _, row := range resp.Values {
          fmt.Printf("%s, %s\n", row[0], row[1])
       }
    }

}
```

Replace the `spreadsheetId` constant with the ID of your target spreadsheet.

This code authenticates with the Google Sheets API using the service account key and reads the data from cells A2 and B2
in the `Sheet1` sheet of the target spreadsheet.

Update the dependencies and run the program:

```bash
go mod tidy
go run main.go
```

The result should look like:

```text
Date, Value:
2024-12-16 10:00:00, 10
```

To write data to a Google Sheets spreadsheet, use the `spreadsheets.Values.Update` and `Spreadsheets.BatchUpdate`
methods. For example, the following modified code inserts a new row above other rows with the current date and an
incremented value:

{{< gist getvictor 5c7fa2770089755066cde5ab2c772cca >}}

We can review the spreadsheet to verify that our code added the new row.

{{< figure src="google-sheets-with-new-row.png" alt="Google Sheets spreadsheet with two rows containing populated Date and Value columns.">}}

## Next steps

- [Automate tracking of engineering metrics](../track-engineering-metrics/).

## Further reading

- Recently, we covered [how to set up a remote development environment](../remote-development-environment/).
- Previously, we showed [how to build a webhook flow with Tines](../webhook-flow-with-tines/).

## Watch how to edit Google Sheets spreadsheet using the API

{{< youtube J2UEYjQVhZ8 >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
