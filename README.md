# PetSafe API Documentation

This repository contains a Postman collection that reverse-engineers key endpoints used by the PetSafe mobile application. The requests cover authentication with AWS Cognito plus the major product domains (Smart Feeders, ScoopFree litter boxes, SmartDoor devices, pet profiles, and sharing metadata). The notes below translate the raw Postman definitions into human-readable reference material so you can explore the APIs without opening Postman.

All request URLs in the collection rely on two base variables:

- `{{cognito_base_url}}` – the AWS Cognito Identity Provider endpoint, e.g. `https://cognito-idp.<region>.amazonaws.com/`.
- `{{petsafe_base_url}}` – the PetSafe API root, which is concatenated with the product path segment (for example `https://api.petsafe.net/`). The collection appends product namespaces such as `smart-feed`, `scoopfree`, `smartdoor`, `pets`, or `directory` to this base.

Additional placeholders (for things like device identifiers or payload values) appear in each request and must be replaced before sending. The sections below list the parameters per endpoint.

## Authentication (AWS Cognito)

PetSafe uses a two-step custom authentication flow. Each request must include the AWS-specific headers shown below.

### Request Login Code
- **Method & URL:** `POST {{cognito_base_url}}`
- **Headers:**
  - `Content-Type: application/x-amz-json-1.1`
  - `X-Amz-Target: AWSCognitoIdentityProviderService.InitiateAuth`
- **Body:**
  ```json
  {
    "ClientId": "{{client_id}}",
    "AuthFlow": "CUSTOM_AUTH",
    "AuthParameters": {
      "USERNAME": "{{email}}",
      "AuthFlow": "CUSTOM_CHALLENGE"
    }
  }
  ```
- **Example response:**
  ```json
  {
    "ChallengeName": "CUSTOM_CHALLENGE",
    "ChallengeParameters": {
      "USERNAME": "example-value",
      "attempts": "1"
    },
    "Session": "sessionid"
  }
  ```
- **Notes:** Initiates login and triggers a one-time code to the registered email address. Capture the returned `Session` token for the next step.

### Exchange Code for Tokens
- **Method & URL:** `POST {{cognito_base_url}}`
- **Headers:**
  - `Content-Type: application/x-amz-json-1.1`
  - `X-Amz-Target: AWSCognitoIdentityProviderService.RespondToAuthChallenge`
- **Body:**
  ```json
  {
    "ClientId": "{{client_id}}",
    "ChallengeName": "{{challenge_name}}",
    "Session": "{{session}}",
    "ChallengeResponses": {
      "USERNAME": "{{username}}",
      "ANSWER": "{{verification_code}}"
    }
  }
  ```
- **Example response:**
  ```json
  {
    "AuthenticationResult": {
      "AccessToken": "accesstoken",
      "ExpiresIn": 3600,
      "IdToken": "idtoken",
      "RefreshToken": "refreshtoken",
      "TokenType": "Bearer"
    },
    "ChallengeParameters": {}
  }
  ```
- **Notes:** Returns the Cognito `AccessToken`, `IdToken`, and `RefreshToken`. Use the `AccessToken` as a `Bearer` token in subsequent PetSafe API calls (add it manually to the `Authorization` header – the collection does not set it automatically).

### Refresh Tokens
- **Method & URL:** `POST {{cognito_base_url}}`
- **Headers:**
  - `Content-Type: application/x-amz-json-1.1`
  - `X-Amz-Target: AWSCognitoIdentityProviderService.InitiateAuth`
- **Body:**
  ```json
  {
    "ClientId": "{{client_id}}",
    "AuthFlow": "REFRESH_TOKEN_AUTH",
    "AuthParameters": {
      "REFRESH_TOKEN": "{{refresh_token}}"
    }
  }
  ```
- **Example response:**
  ```json
  {
    "AuthenticationResult": {
      "AccessToken": "accesstoken",
      "ExpiresIn": 3600,
      "IdToken": "idtoken",
      "TokenType": "Bearer"
    },
    "ChallengeParameters": {}
  }
  ```
- **Notes:** Supply the saved `RefreshToken` to fetch a new short-lived access/id token pair.

## Smart Feeders (Smart Feed)

All Smart Feed endpoints sit under `{{petsafe_base_url}}smart-feed/…` and require the feeder's `thingName` for device-specific calls.

### List Feeders
- **Method & URL:** `GET {{petsafe_base_url}}smart-feed/feeders`
- **Headers:** `Accept: application/json`
- **Notes:** Returns every Smart Feed linked to the authenticated account. The Postman collection does not include an example payload.

### Get Feeder Details
- **Method & URL:** `GET {{petsafe_base_url}}smart-feed/feeders/{{smartfeed_thing_name}}/`
- **Headers:** `Accept: application/json`
- **Notes:** Retrieves the reported state document for a specific feeder. No saved example response is present.

### Feed Now
- **Method & URL:** `POST {{petsafe_base_url}}smart-feed/feeders/{{smartfeed_thing_name}}/meals`
- **Body:**
  ```json
  {
    "amount": {{smartfeed_amount}},
    "slow_feed": {{smartfeed_slow_feed}}
  }
  ```
- **Notes:** Immediately dispenses the requested portion (`amount` is measured in 1/8 cup increments). Set `slow_feed` to `true` for gradual dispensing.

### Get Feeder Messages
- **Method & URL:** `GET {{petsafe_base_url}}smart-feed/feeders/{{smartfeed_thing_name}}/messages`
- **Headers:** `Accept: application/json`
- **Query parameters:** `days={{smartfeed_message_days}}`
- **Notes:** Pulls message history for the feeder, limited to the specified number of days (defaults to 7 in Postman). No example response stored.

### List Feeding Schedules
- **Method & URL:** `GET {{petsafe_base_url}}smart-feed/feeders/{{smartfeed_thing_name}}/schedules`
- **Headers:** `Accept: application/json`
- **Notes:** Returns all configured scheduled meals. The collection does not include an example response.

### Create Feeding Schedule
- **Method & URL:** `POST {{petsafe_base_url}}smart-feed/feeders/{{smartfeed_thing_name}}/schedules`
- **Body:**
  ```json
  {
    "time": "{{smartfeed_schedule_time}}",
    "amount": {{smartfeed_schedule_amount}}
  }
  ```
- **Notes:** Schedules a new meal at 24-hour `HH:MM`. No example response provided.

### Update Feeding Schedule
- **Method & URL:** `PUT {{petsafe_base_url}}smart-feed/feeders/{{smartfeed_thing_name}}/schedules/{{smartfeed_schedule_id}}`
- **Body:**
  ```json
  {
    "time": "{{smartfeed_schedule_time}}",
    "amount": {{smartfeed_schedule_amount}}
  }
  ```
- **Notes:** Adjusts the time or amount for an existing schedule entry.

### Delete Feeding Schedule
- **Method & URL:** `DELETE {{petsafe_base_url}}smart-feed/feeders/{{smartfeed_thing_name}}/schedules/{{smartfeed_schedule_id}}`
- **Notes:** Removes the chosen schedule.

### Delete All Feeding Schedules
- **Method & URL:** `DELETE {{petsafe_base_url}}smart-feed/feeders/{{smartfeed_thing_name}}/schedules`
- **Notes:** Clears every scheduled meal for the feeder.

### Pause or Resume All Schedules
- **Method & URL:** `PUT {{petsafe_base_url}}smart-feed/feeders/{{smartfeed_thing_name}}/settings/paused`
- **Body:**
  ```json
  {
    "value": {{smartfeed_pause_value}}
  }
  ```
- **Notes:** Set `value` to `true` to pause all schedules or `false` to resume them.

### Toggle Child Lock
- **Method & URL:** `PUT {{petsafe_base_url}}smart-feed/feeders/{{smartfeed_thing_name}}/settings/child_lock`
- **Body:**
  ```json
  {
    "value": {{smartfeed_child_lock}}
  }
  ```
- **Notes:** Enables/disables the feeder's physical button lock.

### Toggle Slow Feed Setting
- **Method & URL:** `PUT {{petsafe_base_url}}smart-feed/feeders/{{smartfeed_thing_name}}/settings/slow_feed`
- **Body:**
  ```json
  {
    "value": {{smartfeed_slow_feed_setting}}
  }
  ```
- **Notes:** Globally enables or disables slow-feed mode for scheduled meals.

## ScoopFree Litter Boxes

Endpoints are rooted at `{{petsafe_base_url}}scoopfree/product/product`. Use the litter box's `thingName` for detail operations.

### List Litter Boxes
- **Method & URL:** `GET {{petsafe_base_url}}scoopfree/product/product`
- **Headers:** `Accept: application/json`
- **Notes:** Lists every ScoopFree unit on the account. No example response stored.

### Get Litter Box Details
- **Method & URL:** `GET {{petsafe_base_url}}scoopfree/product/product/{{scoopfree_thing_name}}/`
- **Headers:** `Accept: application/json`
- **Notes:** Returns the latest shadow document for the selected litter box. No example saved.

### Trigger Rake Now
- **Method & URL:** `POST {{petsafe_base_url}}scoopfree/product/product/{{scoopfree_thing_name}}/rake-now`
- **Body:**
  ```json
  {}
  ```
- **Notes:** Starts an immediate raking cycle.

### Reset Rake Counter
- **Method & URL:** `PATCH {{petsafe_base_url}}scoopfree/product/product/{{scoopfree_thing_name}}/shadow`
- **Body:**
  ```json
  {
    "rakeCount": {{scoopfree_rake_count}}
  }
  ```
- **Notes:** Overwrites the reported rake counter.

### Set Rake Delay
- **Method & URL:** `PATCH {{petsafe_base_url}}scoopfree/product/product/{{scoopfree_thing_name}}/shadow`
- **Body:**
  ```json
  {
    "rakeDelayTime": {{scoopfree_rake_delay}}
  }
  ```
- **Notes:** Updates the delay (in minutes) before a rake cycle begins.

### Get Litter Box Activity
- **Method & URL:** `GET {{petsafe_base_url}}scoopfree/product/product/{{scoopfree_thing_name}}/activity`
- **Headers:** `Accept: application/json`
- **Notes:** Fetches recent activity log entries (no saved example).

### Update Arbitrary Setting
- **Method & URL:** `PATCH {{petsafe_base_url}}scoopfree/product/product/{{scoopfree_thing_name}}/settings`
- **Body:**
  ```json
  {
    "{{scoopfree_setting_key}}": {{scoopfree_setting_value}}
  }
  ```
- **Notes:** Generic settings patcher. Wrap string values in quotes manually when substituting environment variables.

## SmartDoor Devices

SmartDoor calls live under `{{petsafe_base_url}}smartdoor/product/product`. The collection includes rich example payloads for these endpoints.

### List SmartDoors
- **Method & URL:** `GET {{petsafe_base_url}}smartdoor/product/product`
- **Description:** Enumerates SmartDoor devices for the logged-in household.
- **Example response (truncated):**
  ```json
  {
    "data": [
      {
        "productType": "smartdoor",
        "thingName": "example-value",
        "schedules": [
          {
            "scheduleId": "example-value",
            "title": "Event",
            "startTime": "00:00",
            "access": 3,
            "dayOfWeek": "1111111"
          }
        ],
        "shadow": {
          "state": {
            "reported": {
              "door": {
                "mode": "MANUAL_UNLOCKED",
                "latchState": "UNLOCKED"
              },
              "power": {
                "hasAdapter": true,
                "batteryLevel": 100
              }
            }
          }
        }
      }
    ]
  }
  ```

### Get SmartDoor Details
- **Method & URL:** `GET {{petsafe_base_url}}smartdoor/product/product/{{smartdoor_thing_name}}/`
- **Description:** Retrieves schedules and shadow state for a single SmartDoor.
- **Example response (truncated):**
  ```json
  {
    "data": {
      "productType": "smartdoor",
      "thingName": "example-value",
      "schedules": [
        {
          "scheduleId": "example-value",
          "title": "Cats in at night",
          "access": 2
        }
      ],
      "shadow": {
        "state": {
          "reported": {
            "door": {
              "mode": "MANUAL_UNLOCKED",
              "latchState": "UNLOCKED"
            }
          }
        }
      }
    }
  }
  ```

### Get SmartDoor Preferences
- **Method & URL:** `GET {{petsafe_base_url}}preferences/product/smartdoor/{{smartdoor_thing_name}}/`
- **Headers:** `Accept: application/json`
- **Example response (truncated):**
  ```json
  {
    "data": {
      "thingName": "example-value",
      "productType": "smartdoor",
      "preferenceData": {
        "friendlyName": "Pet door",
        "tz": "Europe/London"
      }
    }
  }
  ```
- **Notes:** Returns user-editable preferences such as the displayed device name and configured timezone.

### Get SmartDoor Activity
- **Method & URL:** `GET {{petsafe_base_url}}scoopfree/product/product/{{scoopfree_thing_name}}/activity`
- **Description:** The collection maps SmartDoor activity to the same path shape used by ScoopFree. It returns timestamped `code` entries describing schedule actions.
- **Example response (truncated):**
  ```json
  {
    "data": [
      {
        "thingName": "example-value",
        "code": "SCHEDULE_STARTED",
        "payload": {
          "title": "Cats in at night",
          "scheduleId": "example-value"
        },
        "timestamp": "2025-09-23T18:30:43.845Z"
      }
    ]
  }
  ```

### Get SmartDoor Schedules
- **Method & URL:** `GET {{petsafe_base_url}}smartdoor/product/schedules?thingName={{smartdoor_thing_name}}`
- **Headers:** `Accept: application/json`
- **Example response (truncated):**
  ```json
  {
    "data": [
      {
        "scheduleId": "example-value",
        "title": "Dogs in and out",
        "startTime": "00:00",
        "dayOfWeek": "1111111",
        "access": 3,
        "isEnabled": true
      }
    ]
  }
  ```
- **Notes:** Lists every schedule associated with the door, including pet assignments, enabled flags, and next/previous action timestamps.

### Get Override Schedules
- **Method & URL:** `GET {{petsafe_base_url}}smartdoor/product/override/schedules/{{smartdoor_thing_name}}`
- **Headers:** `Accept: application/json`
- **Example response:**
  ```json
  {
    "data": {
      "thingName": "example-value",
      "access": 0
    }
  }
  ```
- **Notes:** Retrieves the currently applied temporary override access level (0 = locked, 3 = unlocked for all pets).

### Lock SmartDoor
- **Method & URL:** `PATCH {{petsafe_base_url}}smartdoor/product/product/{{smartdoor_thing_name}}/shadow`
- **Body:**
  ```json
  {
    "door": {
      "mode": "MANUAL_LOCKED"
    }
  }
  ```
- **Example response (truncated):**
  ```json
  {
    "data": {
      "state": {
        "desired": {
          "door": {
            "mode": "MANUAL_LOCKED"
          }
        },
        "reported": {
          "door": {
            "mode": "MANUAL_UNLOCKED",
            "latchState": "UNLOCKED"
          }
        }
      }
    }
  }
  ```

### Unlock SmartDoor
- **Method & URL:** `PATCH {{petsafe_base_url}}smartdoor/product/product/{{smartdoor_thing_name}}/shadow`
- **Body:**
  ```json
  {
    "door": {
      "mode": "MANUAL_LOCKED"
    }
  }
  ```
- **Example response (truncated):**
  ```json
  {
    "data": {
      "state": {
        "reported": {
          "door": {
            "mode": "MANUAL_LOCKED",
            "latchState": "LOCKED"
          }
        }
      }
    }
  }
  ```
- **Notes:** The Postman collection stores the same payload as the lock request. In practice you should send `"mode": "MANUAL_UNLOCKED"` to release the latch.

### Smart Mode
- **Method & URL:** `PATCH {{petsafe_base_url}}smartdoor/product/product/{{smartdoor_thing_name}}/shadow`
- **Body:**
  ```json
  {
    "door": {
      "mode": "SMART"
    }
  }
  ```
- **Example response (truncated):**
  ```json
  {
    "data": {
      "state": {
        "desired": {
          "door": {
            "mode": "MANUAL_LOCKED"
          }
        },
        "reported": {
          "door": {
            "mode": "MANUAL_UNLOCKED",
            "latchState": "UNLOCKED"
          }
        }
      }
    }
  }
  ```


### Update SmartDoor Friendly Name
- **Method & URL:** `PATCH {{petsafe_base_url}}preferences/product/smartdoor/{{smartdoor_thing_name}}`
- **Body:**
  ```json
  {
    "friendlyName": "Pet door"
  }
  ```
- **Example response (truncated):**
  ```json
  {
    "data": {
      "preferenceData": {
        "friendlyName": "Pet door",
        "tz": "Europe/London"
      }
    }
  }
  ```
- **Notes:** Updates the label shown in the mobile app without altering schedules or access rules.

### Update SmartDoor Timezone
- **Method & URL:** `PATCH {{petsafe_base_url}}preferences/product/smartdoor/{{smartdoor_thing_name}}`
- **Body:**
  ```json
  {
    "tz": "Europe/London"
  }
  ```
- **Example response:**
  ```json
  {
    "data": {
      "preferenceData": {
        "friendlyName": "Pet door",
        "tz": "Europe/London"
      }
    }
  }
  ```
- **Notes:** Sets the timezone that schedule start times are interpreted against.

### Set Final Act to Unlocked
- **Method & URL:** `PATCH {{petsafe_base_url}}smartdoor/product/product/{{smartdoor_thing_name}}/shadow`
- **Body:**
  ```json
  {
    "power": {
      "finalAct": "UNLOCKED"
    }
  }
  ```
- **Example response (truncated):**
  ```json
  {
    "data": {
      "state": {
        "desired": {
          "power": {
            "finalAct": "UNLOCKED"
          }
        }
      }
    }
  }
  ```
- **Notes:** Controls how the door should behave when power is critically low or restored.

### Set Final Act to Locked
- **Method & URL:** `PATCH {{petsafe_base_url}}smartdoor/product/product/{{smartdoor_thing_name}}/shadow`
- **Body:**
  ```json
  {
    "power": {
      "finalAct": "LOCKED"
    }
  }
  ```
- **Example response (truncated):**
  ```json
  {
    "data": {
      "state": {
        "desired": {
          "power": {
            "finalAct": "LOCKED"
          }
        }
      }
    }
  }
  ```
- **Notes:** Sets the door to lock itself during the final act condition (for example when batteries are depleted).

### Apply Override Schedule Access
- **Method & URL:** `PATCH {{petsafe_base_url}}smartdoor/product/override/schedules`
- **Body:**
  ```json
  {
    "thingName": "example-value",
    "access": 0
  }
  ```
- **Example response:**
  ```json
  {
    "data": {
      "thingName": "example-value",
      "access": 0
    }
  }
  ```
- **Notes:** Temporarily overrides the standard schedules with the supplied access level until cleared or expired.

### Create or Update SmartDoor Schedule
- **Method & URL:** `POST {{petsafe_base_url}}smartdoor/product/schedules`
- **Body:**
  ```json
  {
    "title": "My schedule name",
    "thingName": "example-value",
    "isEnabled": true,
    "dayOfWeek": "0001010",
    "startTime": "10:02",
    "access": 3,
    "petIds": [
      "example-value"
    ]
  }
  ```
- **Example response (truncated):**
  ```json
  {
    "data": [
      {
        "scheduleId": "example-value",
        "title": "Dogs in and out",
        "dayOfWeek": "1111111"
      }
    ]
  }
  ```
- **Notes:** Creates a new schedule or updates an existing entry when a `scheduleId` is included in the payload.

### Delete SmartDoor Schedule
- **Method & URL:** `DELETE {{petsafe_base_url}}smartdoor/product/schedules/{{smartdoor_schedule_id}}`
- **Body:**
  ```json
  {
    "title": "My schedule name",
    "thingName": "example-value",
    "isEnabled": true,
    "dayOfWeek": "1110101",
    "startTime": "10:02",
    "access": 3,
    "petIds": [
      "example-value"
    ]
  }
  ```
- **Example response (truncated):**
  ```json
  {
    "data": [
      {
        "scheduleId": "example-value",
        "title": "Dogs in and out",
        "dayOfWeek": "1111111"
      }
    ]
  }
  ```
- **Notes:** Removes the targeted schedule and returns the remaining entries for confirmation.

## Pets and Directory APIs

### List Pets
- **Method & URL:** `GET {{petsafe_base_url}}pets/pets`
- **Headers:** `Accept: application/json`
- **Example response (truncated):**
  ```json
  {
    "data": [
      {
        "petId": "example-value",
        "profile": {
          "name": "Tailwag",
          "petType": "Cat",
          "weight": 2.92
        },
        "technologies": [
          {
            "tagId": "example-value",
            "technology": "MICROCHIP"
          }
        ]
      }
    ]
  }
  ```

### List Pet Products
- **Method & URL:** `GET {{petsafe_base_url}}directory/petProduct`
- **Headers:** `Content-Type: application/json`
- **Query parameters:** `petId={{petId}}`
- **Example response:**
  ```json
  {
    "data": [
      {
        "productType": "smartdoor_1_large",
        "petId": "example-value",
        "productId": "example-value"
      }
    ]
  }
  ```

### Product Sharing Metadata
- **Method & URL:** `GET {{petsafe_base_url}}directory/product-sharing`
- **Headers:**
  - `Accept-Encoding: gzip`
  - `Content-Type: application/json`
  - `Cache-Control: no-cache`
  - `User-Agent: my PetSafe Code Version: 132 App Version: 1.19.1 Stage: Production Device: Google sdk_gphone64_x86_64 Android: Android SDK: 36 (16)`
- **Example response:**
  ```json
  {
    "data": [
      {
        "email": "example@example.com",
        "ownerId": "example-value",
        "userId": "example-value",
        "nickname": "example@example.com",
        "status": "shared"
      }
    ]
  }
  ```


### Account Details
- **Method & URL:** `GET {{petsafe_base_url}}directory/account`
- **Headers:**
  - `Accept-Encoding: gzip`
  - `Content-Type: application/json`
  - `Cache-Control: no-cache`
  - `User-Agent: my PetSafe Code Version: 132 App Version: 1.19.1 Stage: Production Device: Google sdk_gphone64_x86_64 Android: Android SDK: 36 (16)`
- **Example response (truncated):**
  ```json
  {
    "data": {
      "email": "example@example.com",
      "userId": "example-value",
      "userData": {
        "display12HourTime": true,
        "receiveNotifications": true,
        "region": "us-east-1"
      },
      "products": [
        {
          "entityData": {
            "friendlyName": "Pet door",
            "productType": "smartdoor"
          }
        }
      ]
    }
  }
  ```
- **Notes:** Returns account profile settings along with linked product and pet identifiers.

## Usage Tips

- Replace every `{{placeholder}}` with your own value or an environment variable in your HTTP client.
- Supply the Cognito `AccessToken` via `Authorization: Bearer <token>` on every PetSafe API call. The Postman definitions assume you add this header manually or through a collection-level script.
- Many endpoints operate on AWS IoT "shadow" documents. Patch requests write to the `desired` state while GET calls surface the current `reported` state.
- The collection stores example responses for SmartDoor, pet, and sharing endpoints. Others return live data only, so adjust the documentation according to actual responses you receive.
