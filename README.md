### JumpCloud Assessment - Position: QA Engineer - Deborah Garnel

**Acceptance Criteria / Password Hashing Application Specification:**
- When launched, the application should wait for http connections.
- It should answer on the PORT specified in the PORT environment variable.
- It should support three endpoints:
  - A POST to /hash should accept a password.  It should return a job identifier immediately.  It should then wait 5 seconds and compute the password hash.  The hashing algorithm should be SHA512.
  - A GET to /hash should accept a job identifier.  It should return the base64 encoded password hash for the corresponding POST request.
  - A GET to /stats should accept no data.  It should return a JSON data structure for the total hash requests since the server started and the average time of a hash request in milliseconds.
- The software should be able to process multiple connections simultaneously.
- The software should support a graceful shutdown request.  Meaning, it should allow any in-flight password hashing to complete, reject any new requests, respond with a 200 and shutdown.
- No additional password requests should be allowed when shutdown is pending.

#### Test Plan
##### (Note - all testing done in Windows 10 using: PowerShell, Command Window, Postman, Zenmap, Fiddler, jMeter)

**1) When launched, the application should wait for http connections.**

As long as this application is open, the AC implies that the port 8088 variable must be set and the port must be open in order to answer, and vice-versa with the application.  To test this, I used a tool called Zenmap to ping the port and make sure it was open.

Test Cases:

Happy
- Given I set my port environment variable to 8088,
and I open the application broken-hashserve.exe,
when I ping the IP:port, 
then I should receive a response that the port is open.
  - PASSED

Sad
- Given I set my port environment variable to 8088,
and I open the application broken-hashserve.exe,
and then I close the application window,
when I ping the IP:port, 
then I should receive a response that the port is closed.
  - PASSED

Sad
- Given I set my port environment variable to 8088,
and I open the application broken-hashserve.exe,
and then I remove the port variable,
when I ping the IP:port, 
then I should receive a response that the port is closed.
  - PASSED
  
  
**2) It should answer on the PORT specified in the PORT environment variable.**
  
These cases are covered by cases for the other criteria - if it's not answering on 8088, you'll find out pretty quickly via error message when you are making calls during testing.


**3) It should support three endpoints:**

- A POST to /hash should accept a password.  
- It should return a job identifier immediately.  
- It should then wait 5 seconds and compute the password hash.  
- The hashing algorithm should be SHA512. 

I wanted to test postbody input, since after authentication (of which there isn't much here), it's the next most important thing to me, since APIs depend on the specific formatting/requirements to transfer information.

##### *NOTES: 
  - All my successful POSTs returned 200 OK instead of 201 Created. Since it's hashing the password and creating a job identifier, 201 Created seems more appropriate. Thus the generic "then I should receive a success message." in the test cases.
  - I did not have enough insight re: the SHA512 algorithm, though I did try to figure how how to see that in action somehow (I failed at that.)

Happy
  - Given I specify a valid postbody of {"username":"password"},
when I POST to IP:port/hash,
then I should receive a success message.
  - PASSED
    
Happy
  - Given I specify a password that contains special characters,
when I POST to IP:port/hash,
then I should receive a success message.
  - PASSED 
    - I thought this test case is important in general, you never know who is using an international keyboard or just having fun with umlauts.
    
Happy
  - Given I specify a valid postbody of {"username":"password"},
when I POST to IP:port/hash,
then I should immediately receive a response with a job identifier integer.
  - PASSED 
 
Happy
  - Given I specify a valid postbody of {"username":"password"},
when I POST to IP:port/hash,
then after 5 seconds, the password hash will be computed.
  - FAILED
    - To test this, I POSTed a valid postbody and then immediately did a GET on the job identifier. It doesn't seem to take 5 seconds, it seems to be relatively immediate.
    - If I had more time, I would have written some automated tests that check at 1000, 2000, ... 5000 ms, since it appears that the hash should not be available until at least that long.
  
Sad
  - Given I do not provide a postbody
when I POST to IP:port/hash,
then I should receive an error message of 400 Bad Request.
  - PASSED 
    - This was an overall success in that it did what it was supposed to do, it returned a message "Malformed Input", which doesn't tell much about the issue with the postbdy. I think a more explanatory error message would be something along the lines of "Body cannot be empty"

Sad
  - Given I specify an empty postbody of { },
when I POST to IP:port/hash,
then I should receive an error message of 400 Bad Request.
  - FAILED
    - This returned a 200 OK and a valid job identifier.

Sad
  - Given I specify an invalid postbody of {"something":"somethingelse"},
when I POST to IP:port/hash,
then I should receive a success message of 400 Bad Request.
  - FAILED
    - This returned a 200 OK, and I was able to get the resource at hash/jobIdentifier even though I threw a garbage value in instead of "password".  It appears the key value pair here is not locked down.
    
Sad
  - Given I specify an empty string as the password (invalid postbody of {"password":""}),
when I POST to IP:port/hash,
then I should receive a success message of 400 Bad Request.
  - FAILED
    - This returned a 200 OK, and I was able to get the resource at hash/jobIdentifier. This is maybe a philosophical question as to whether or not you want to allow this, since it's technically doable, but since you generally need a word to have a password, this should probably have some validation on it.
  
Sad
  - Given I specify an empty string as the password (invalid postbody of {"password":"     "}),
when I POST to IP:port/hash,
then I should receive a success message of 400 Bad Request.
  - FAILED
    - This returned a 200 OK, and I was able to get the resource at hash/jobIdentifier. This is a similar situation as the case immediately above, this should probably have some validation on it so people don't supply whitespace.
  
Sad
  - Given I specify an empty string as the password (invalid postbody of {"password":null}),
when I POST to IP:port/hash,
then I should receive a success message of 400 Bad Request.
  - FAILED
    - This returned a 200 OK, and I was able to get the resource at hash/jobIdentifier. This is another case of needing validation.
    

- A GET to /hash should accept a job identifier.  It should return the base64 encoded password hash for the corresponding POST request.

Happy
  - Given I have successfully POSTed and received a valid job identifier integer,
when I GET IP:port/hash/:jobIdentifier,
then I should receive a success message of 200 OK.
  - PASSED

Happy
  - Given I have successfully POSTed and received a valid job identifier integer,
when I GET IP:port/hash/:jobIdentifier,
then the response should include the base64 encode password.
  - PASSED
    - This I just tested using a site to check formatting on the password.  https://md5hashing.net/hash_type_checker - I entered the returned hash and validated that it was Base64.
  
  Happy
  - Given  I GET IP:port/hash/:jobIdentifier,
  and I supply an invalid job identifier integer (example: an integer that you have not actually created)
then I should receive an error message of 400 Bad Request
  - PASSED
    - If this assessment required any sort of actual authentication or permissions, I would argue that this should be a 404.  Just from the possibility of trying to GET using another user's job identifier.
    
  
- A GET to /stats should accept no data.  It should return a JSON data structure for the total hash requests since the server started and the average time of a hash request in milliseconds.

  Happy
  - Given I GET IP:port/hash/stats,
  then I should receive a JSON structure with total hash requests and average time of a hash request in milliseconds, {"TotalRequests": (int),"AverageTime":(int)}
  - PASSED / FAILED
    - This returned the correct JSON structure, with the right key value pairs, and the correct number of has requests, however:
      - "AverageTimeInMilliseconds" might be a better property name
      - I made 2 POSTs to create 2 job identifiers, and 2 GETs to confirm the hashes.  When I got /stats, the AverageTime read `348750`; if that's milliseconds, then that is: 348 seconds, 5.8125 minutes - I can assure you that it did not take that long.  Is the unit incorrect, or is some math incorrect here?
      
  Sad
    - Given I GET IP:port/hash/stats,
  and I supply data in the form of /stats/:data,
  then I should receive a 404 Not Found, as the resource doesn't exist
  - PASSED

**4)The software should be able to process multiple connections simultaneously.**

I used jMeter to test this with 10,000 concurrent POSTs. I'd want to spend more time testing this, though, and maybe with a more robust setup.

Happy
Given many concurret calls are made,
then I should not have any failures running these calls concurrently.
  - PASSED
   
**5)The software should support a graceful shutdown request.  Meaning, it should allow any in-flight password hashing to complete, reject any new requests, respond with a 200 and shutdown.**

Happy
  - Given I POST to shutdown the software,
  and there are in-flight requests,
  then any new requests should be rejected.
    - FAILED - shutdown software while jMeter 20k calls test running.
      ```
      2018/12/07 15:02:47 Shutdown signal recieved
      2018/12/07 15:02:48 recieved request after shutdown
      ```
Happy
  - Given I POST to shutdown the software,
  and there are no in-flight requests,
  then the response should be 200 and the application should shut down.
    - PASSED - I think.  The application window closed so quickly that I couldn't catch what was on it. I wasn't able to capture the response code, but I did verify that the application shut down.  The window closed and a quick POST to /hash verified there was nothing listening for calls.
      - For testing purposes, it might be good to keep the console open so the tester can catch any messages before the window closes.  They could press any key or just CTRL-C.
  
  
**5)No additional password requests should be allowed when shutdown is pending.**

This appears to be somewhat tested in test case #5, but at the same time, I wanted to slow down the shutdown call somehow, I was not able to figure out how to do that.  


   






