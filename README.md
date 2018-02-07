# JumpCloudExercise

# JumpCloud QA exercise

Approach: I wrote out the basic test cases for the requirements mentioned directly or implied in the application specification. 
The next section below has the test run results and bugs found.
Further tests not included here would be:
- Verify in the database that the in-flight requests during a shutdown request are in fact encoded and saved, after the identifier is returned.
- Load testing the endpoint more robustly rather than just verifying it can handle some concurrent requests.
- Password complexity would probably be included in a real app, which should be tested as well.
- Verify other endpoints are not supported with unexpected database changes - PUT, DELETE, etc
- If there was a requirement for the number of requests that can be handled at a time, there would be boundary test cases for success below the limit and failure above it

## **Test plan**

  Requirement: When launched, the application should wait for http connections
  - Test case 1:
  - Step 1: Launch application
  - Result 1: Application launches and waits
  - Step 2: Send http request
  - Result 2: Application responds and remains open

  Requirement: It should answer on the PORT specified in the PORT environment variable.
  - Test case 2:
  - Launch application with port variable set to 8088
  - Step 1: Make request with port 8088. e.g. curl -X POST -H "application/json" -d '{"password":"angrymonkey"}'             http://127.0.0.1:8088/hash
  - Result 1: Application responds
  - Step 2: Make request with different port
  - Result 2: Application returns connection error

  Requirement: A POST to /hash should accept a password
  - Test case 3:
  - Launch application with port variable set to 8088
  - Step 1: Post with -d '{"password":"value"}
  - Result 1: Application responds
  - Step 2: Post without value -d '{"password":""}
  - Result 2: Application returns 422, password value required
  - Step 3: Post with different key -d '{"passphrase":"value
  - Result 3: Application returns 400, invalid key
	- Step 4: Post with additional key value pairs
  - Result 4: Application returns 400, invalid key

  Requirement: A POST to /hash should return a job identifier immediately
  - Test case 4:
  - Launch application with port variable set to 8088
  - Step 1: Post with -d '{"password":"value"}
  - Result 1: Application response is immediate and includes a job identifier
  - Step 2: After 5 seconds, Get /hash/<identifier>
  - Result 2: Get responds verifying identifier was saved and is valid

  Requirement: A POST to /hash should wait 5 seconds and compute the password hash
  - Test case 5:
  - Launch application with port variable set to 8088 and Post with -d '{"password":"value"}
  - Step 1: Immediately Get /hash/<identifier>
  - Result 1: Application returns 404, hash not created yet
  - Step 2: After 4 seconds, Get /hash/<identifier>
  - Result 2: Application returns 404, hash not created yet
  - Step 3: After 5 seconds, Get /hash/<identifier>
  - Result 3: Application returns encoded password

  Requirement: The hashing algorithm should be SHA512
  - Test case 6:
  - Launch application with port variable set to 8088 and Post with -d '{"password":"value"}
  - encode the password value using SHA512 and Base64, e.g. https://www.movable-type.co.uk/scripts/sha512.html
  - Step1: After 5 seconds, Get /hash/<identifier>
  - Result 1: Application returns encoded password matching precalculated value

  Requirement: A GET to /hash should accept a job identifier
  - Test case 7:
  - Launch application with port variable set to 8088, Post with -d '{"password":"value"}, get the identifier from the response
  - Step 1: Get to /hash/<identifier>
  - Result 1: Application returns encoded password
  - Step 2: Get to /hash without identifier
  - Result 2: Application returns 400 bad path 
  - Step 3: Get to /hash/<identifier> with invalid identifier value
  - Result 3: Application returns 404 

  Requirement: It should return the base64 encoded password hash for the corresponding POST request
  - Test case 8:
  - Launch application with port variable set to 8088 and Post with -d '{"password":"value"}
  - encode the password value using SHA512 and Base64
  - Step1: After 5 seconds, Get /hash/<identifier>
  - Result 1: Application returns encoded password matching precalculated value

  Requirement: A GET to /stats should accept no data
  - Test case 9:
  - Step 1: Get to /stats 
  - Result 1: Application responds with 200 
  - Step 2: Get to /stats/<something>
  - Result 2: Application returns 400, invalid path

  Requirement: A GET to /stats should return a JSON data structure
  - Test case 10:
  - Step 1: Get to /stats 
  - Result 1: Response is JSON

  Requirement: A GET to /stats should return the total hash requests since the server started and the average time of a hash request in milliseconds
  - Test case 11:
  - Execute after the other tests, so as to have data 
  - Step 1: Get to /stats
  - Result 1: Response includes "TotalRequests" and "AverageTime" keys
  - Result 2: TotalRequests value is correct for the given session
  - Result 3: AverageTime is in milliseconds and looks correct for the given session

  Requirement: The software should be able to process multiple connections simultaneously
  - Test case 12:
  - Step 1: Send 8 Post requests in parallel (in bash, run same command with ' &' at the end of each curl, then type 'wait' to execute them)
  - Result 1: Each request is handled successfully
  - Step 2: Send multiple Get requests in parallel
  - Result 2: Each request is handled successfully

  Requirement: Application accepts shutdown request
  - Test case 13:
  - Launch application with port variable set to 8088
  - Step 1: POST -d "shutdown"
  - Result 1: Application returns a 200 response
  - Result 2: Application shuts down 

  Requirement: Application completes in-flight requests before shutting down
  - Test case 14:
  - Launch application with port variable set to 8088 and send a Post
  - Step 1: POST -d ‘shutdown’ less than 5 seconds after the Post request, save the identifier
  - Result 1: Application returns a 200 response
  - Result 2: Application encodes the password from the Post
  - Result 3: Application shuts down
  - Step 2: Launch the application and Get the identifier from step 1
  - Result 2: Encoded password is returned, verifying the previous request was completed before shutdown

  Requirement: No additional password requests should be allowed when shutdown is pending
  - Test case 15:
  - Launch application with port variable set to 8088 and send a Post
  - Step 1: POST -d ‘shutdown’ less than 5 seconds after the Post request, save the identifier
  - Result 1: Application returns a 200 response
  - Step 2: Post another request within 5 seconds of the 1st one
  - Result 2: Application returns an error

## **Test run results:**

Test case 1: Pass

Test case 2: Pass

Test case 3: fail
- Issue 1: Empty password values are accepted, like {"password":""}
- Issue 2: Incorrect keys are accepted, like {"whatever":"angrymonkey"}, and additional key value pairs are accepted

Test case 4: fail
- Issue 3: Posting to /hash returns an identifier after 5 seconds instead of immediately

Test case 5: Partial pass 
- Steps 1,2 are blocked by issue 3. Step 3 passes

Test case 6: fail
- Issue 4: The password does not appear to be encoded in SHA512. The result also can't be decoded with Base64, so it is hard to say if the SHA512 part was correct, but the encoded string seems too short for that
	- The password was 'tc6', which has
	- SHA512: a4ff034704d8c091d84b9e2ff9cef21c62321e7ce2cfbb9caeee0f1a762505e60845cf3ec785a302fb5ff9c7baf511aa5da7957e1602be50930ffdae81b83bfa
	- For which Base64 would be: YTRmZjAzNDcwNGQ4YzA5MWQ4NGI5ZTJmZjljZWYyMWM2MjMyMWU3Y2UyY2ZiYjljYWVlZTBmMWE3NjI1MDVlNjA4NDVjZjNlYzc4NWEzMDJmYjVmZjljN2JhZjUxMWFhNWRhNzk1N2UxNjAyYmU1MDkzMGZmZGFlODFiODNiZmE=
	- The app response is: pP8DRwTYwJHYS54v+c7yHGIyHnziz7ucru4PGnYlBeYIRc8+x4WjAvtf+ce69RGqXaeVfhYCvlCTD/2ugbg7+g==
	- It also doesn't match SHA256 or Base32 encoding, which is all the guessing I feel the test case deserves for now.

Test case 7: Partial pass
- Issue 5: (Minor issue): GET with a validly formatted identifier that doesn't exist returns a 400 instead of the more appropriate 404

Test case 8: fail
- Issue 4 (see description above)

Test case 9: Pass

Test case 10: Pass

Test case 11: Partial pass
- Results 1,2 pass, but not Result 3
- Issue 6: The AverageTime value is too large, between 45000 - 70000. The /hash requests are consistently slightly over 5000 ms

Test case 12: Partial pass
- Issue 7: (Minor issue) only POST requests are handled simultaneously, not GET requests. Not sure whether the requirement applies to GETs

Test case 13: Pass

Test case 14: Pass
- Step 2 is blocked, since data doesn't appear to be saved between sessions, so other than getting an identifier, can't verify that the in-flight post request encoded the password before shutting down

Test case 15: Pass
