# Discord Dynamic RPC

Solution for dynamic application names with Discord's RPC functionality.

This project provides a server and API to collect Discord application client IDs
created in the [Discord Developer Portal](https://discord.com/developers/application)
to allow users to have somewhat dynamic text in their status,
e.g. "Listening to One Republic" or "Watching True Detective",
when it is unrealistic to manually create applications for all cases.

The API provides an endpoint for
submitting the client ID of a Discord application
and for retrieval of a Discord application ID for a given text.

## Table of Contents

- [HTTP Endpoints](#http-endpoints)
  - [Submit a Discord application client ID](#submit-a-discord-application-client-id)
    - [Request](#request)
    - [Status codes](#status-codes)
    - [Response body](#response-body)
  - [Search a Discord application ID for a name](#search-a-discord-application-id-for-a-name)
    - [Request](#request-1)
    - [Query parameters](#query-parameters)
    - [Status codes](#status-codes-1)
    - [Response body](#response-body-1)
    - [Example](#example)
  - [Report an inconsistent Discord application name](#report-an-inconsistent-discord-application-name)
    - [Request](#request-2)
    - [Query parameters](#query-parameters-1)
    - [Status codes](#status-codes-2)
    - [Response body](#response-body-2)
- [WebSocket Endpoint](#websocket-endpoint)
    - [Searching a Discord application by name](#searching-a-discord-application-by-name)
- [Useful Discord endpoints](#useful-discord-endpoints)
  - [Verify the name of a Discord application](#verify-the-name-of-a-discord-application)

## HTTP Endpoints

### Submit a Discord application client ID

Use this endpoint to submit a Discord application client ID
together with the expected name of the Discord application.
The name is verified with Discord and then stored in the database.
The application name and the expected name must match exactly,
the comparison is case-sensitive.

#### Request

```json
POST /submit
{
    "discord_application_client_id": "1875476348971658765334",
    "expected_name": "One Republic"
}
```

#### Status codes

- **200** OK
  - The expected name was successfully verified and the client ID was stored.
- **400** Bad Request
  - The Discord application could not be found with the given client ID or
  - The application's name does not have the expected name
- **409** Conflict
  - There are too many registered Discord applications for this name already
- **502** Bad Gateway
  - Failed to verify the Discord application client ID
    with Discord the Discord API
- **500** Internal Server Error
  - A internal error occurred
- **504** Gateway Timeout
  - Verifying the Discord application client ID with the Discord API timed out

#### Response body

The response always contains plain text with a status or error message.

### Search a Discord application ID for a name

#### Request

```
GET /search
```

#### Query parameters

- (string) `name` The required name of the of the Discord application.
  This parameter is required
- (boolean) `case` Whether to only include exact, case-sensitive matches.
  This parameter is optional and true by default

#### Status codes

- **200** OK
  - One or more matches were found
- **400** Bad Request
  - One or more query parameters were missing or had invalid values
- **404** Not Found
  - No matches where found for the given search criteria
- **500** Internal Server Error
  - A internal error occurred

#### Response body

In the case of a 200 status code, the response body is always JSON
with the following structure:

```json
{
    "results": [
        {
            "name": "...",
            "discord_application_id": "...",
            "discord_application_client_id": "...",
            "exact_match": true
        },
        // ...
    ]
}
```

The results always include at least one item.
Exact matches always come first in the results array
and their `exact_match` key is set to true.
There may be multiple exact matches
when there are multiple registered Discord applications with the given name.
The array may contain additional items for inexact matches,
when the `case` query parameter is set to false.

It is recommended to use the Discord endpoint documented below
to verify that the given Discord application has the expected name.
Results with identical `exact_match` value
may be ordered randomly amongst each other
in order to ensure each entry is validated regularly by clients.

For all other status codes,
the response is plain text with a status or error message.

#### Example

```json
GET /get?name=One%20Republic&case=false
200 OK
{
    "results": [
        {
            "name": "One Republic",
            "discord_application_id": "15497836517643166231",
            "discord_application_client_id": "1875476348971658765334",
            "exact_match": true
        },
        {
            "name": "one republic",
            "discord_application_id": "61398719874573113433",
            "discord_application_client_id": "389761896389741983763",
            "exact_match": false
        }
    ]

}
```

### Report an inconsistent Discord application name

When using the Discord endpoint that is documented below
to verify that a Discord application has the correct name,
you may encounter that it has been changed by the owner of that application.

While the server regularly checks that all registered application IDs
still have the expected name,
clients can notify the server of any inconsistencies.
This is encouraged to ensure the quality of the internal database.

#### Request

```
POST /report-inconsistency
```

#### Query parameters

- (string) `discord_application_client_id`
  The Discord application client ID whose name is suspected to be inconsistent.
  This parameter is required

#### Status codes

- **200** OK
  - The report was acknowledged and is being processed
- **400** Bad Request
  - One or more query parameters were missing or had invalid values
- **500** Internal Server Error
  - A internal error occurred

#### Response body

The response always contains plain text with a status or error message.

## WebSocket Endpoint

To open a websocket connection for streaming communication,
make a request to the following endpoint
and upgrade it to a Websocket connection.

```
GET /ws
```

#### Searching a Discord application by name

Send a JSON message with the following format to perform a search:

```json
{
    "id": 1,
    "type": "search",
    "data": {
        "name": "...",
        "case": true
    }
}
```

The `type` key must be `"search"`.

The `id` may be an arbitrary number to identify the request
and it is set in the response so it can be tied to this request.

The keys in `data` have the same meaning
as the query parameters of the `/get` HTTP endpoint.

The server will respond with the following message:

```json
{
    "id": 1,
    "type": "search",
    "code": 200,
    "data": {
        // ...
    }
}
```

The `id` is identical to the `id` of the search request.
For unambiguous matching it is recommended
to increment the ID with every search request.

The `type` key is always `"search"`.

The `code` is identical to the status code of the `/get` HTTP endpoint.

The `data` is identical to the response body of the `/get` HTTP endpoint,
which may be either an object in case of a 200 status code
or a simple status or error message (string) in case of other status codes.

## Useful Discord endpoints

### Verify the name of a Discord application

You should use this endpoint to verify that a Discord application whose application ID or client ID you are using has the correct name,
where `client_id` is the `discord_application_client_id`
retrieved from one of the endpoints above:

```
GET https://discord.com/api/v10/oauth2/applications/{client_id}/rpc
```

The response contains a JSON object with a `name` key
which should match with the name provided by the above endpoints.
The `id` key represents the Discord application ID used for RPC,
which should match the `discord_application_id`
provided by the above endpoints.
