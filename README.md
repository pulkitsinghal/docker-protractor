Dockerfile for [Protractor](http://angular.github.io/protractor/) test execution
================================================================================

Summary
-------

1. Clone this project: `git clone https://github.com/pulkitsinghal/docker-protractor.git`
1. Build a docker image from this project:

    ```
    cd docker-protractor
    docker build -t docker-protractor .
    ```
    * I could make yet another image available on DockerHub but if not enough people benefit from this fork then its not practical. That is the reason I recommend building it yourself. If you hate this because bandwidth or CPU/memory is an issue for you then I recommend reading [this](https://training.shoppinpal.com/solution.html).
1. Switch to the root project directory of your application.
1. When running this docker container:
    1. In your application, create a copy your existing `conf` file and rename it as `protractor.conf.js` because that's the only thing that this image will look for right now, nothing else.
    1. `protractor.conf.js` must be created or updated such that chrome is run with the `no-sandbox` option
    1. You need to start your application and have a `BASEURL` ready for reference so that the protractor tests have something to run against. Let's hope you don't need too many code changes to follow this best practice.
        * If you need to run against another container then there are instructions near the end of this README. Since this project is a fork, I have not tried it myself.
    1. Mount the directory which contains your tests (not your entire application) as a volume under `/project`, examples:

        ```
        # mounts present working directory into container under /project
        # protractor.conf.js should be in `pwd`
        -v `pwd`:/project
        
        # OR, if `protractor.conf.js` is in `tests` folder, then
        -v `pwd`/tests:/project

        # OR, if `protractor.conf.js` is in `tests/e2e` folder, then
        -v `pwd`/tests/e2e:/project
        ```
    1. So you will be running something like:

        ```
        docker run --rm \
          -v <test project location>:/project \
          --env BASEURL=<point at the URL where your application is running>
          docker-protractor
        ```
1. You may `docker-compose.e2e.yml` file to the root directory of your application project to turn your end-2-end tests into a load testing harness!
    1. Sample:
        ```
        version: '2'
        services:
        e2e:
          image: docker-protractor
          environment:
            - BASEURL=http://staging.app.com
          volumes:
            - ./tests/e2e:/project
        ```
    1. You can then start and scale as many e2e tests as you would like:
        ```
        #  run them once:
        docker-compose --file ./docker-compose.e2e.yml up
        #  run them 5 times (in parallel):
        docker-compose --file ./docker-compose.e2e.yml up --scale e2e=5
        #  run them 100 times (in parallel):
        docker-compose --file ./docker-compose.e2e.yml up --scale e2e=5

        # etc.
        # you get the idea ...
        ```

Background
----------
Based on [caltha/protractor](https://bitbucket.org/rkrzewski/dockerfile), this image contains a fully configured environment for running Protractor tests under the Chromium browser.

This version additionally supports linking docker containers together to test software in another container, and passing a custom base URL into your protractor specs so you don't have to hard-code the URL in them. 

Installed software
------------------
   * [Xvfb](http://unixhelp.ed.ac.uk/CGI/man-cgi?Xvfb+1) The headless X server, for running browsers inside Docker
   * [node.js](http://nodejs.org/) The runtime platform for running JavaScript on the server side, including Protractor tests
   * [npm](https://www.npmjs.com/) Node.js package manager used to install Protractor and any specific node.js modules the tests may need
   * [Selenium webdriver](http://docs.seleniumhq.org/docs/03_webdriver.jsp) Browser instrumentation agent used by Protractor to execute the tests
   * [OpenJDK 8 JRE](http://openjdk.java.net/projects/jdk8/) Needed by Selenium
   * [Chromium](http://www.chromium.org/Home) The OSS core part of Google Chrome browser
   * [Protractor](http://angular.github.io/protractor/) An end-to-end test framework for web applications
   * [Supervisor](http://supervisord.org/) Process controll system used to manage Xvfb and Selenium background processes needed by Protractor

Running
-------
In order to run tests from a CI system, execute the following:
```
docker run --rm -v <test project location>:/project mrsheepuk/protractor
```
The container will terminate automatically after the tests are completed.

To run against another container, execute as follows (this example assumes that the image you are testing exposes its web site on port 3000):
```
docker run -d --name=webe2e <image to test>
docker run --rm --link=webe2e:webe2e -v <test project location>:/project --env BASEURL=http://webe2e:3000/ mrsheepuk/protractor
docker rm -f webe2e
```

You can also use the BASEURL variable without container linking, to test any arbitrary web site. If you wish to use the BASEURL functionality, you must use relative URLs within your test specs (e.g. `browser.get("/profile/")` instead of `browser.get("http://test.com/profile/")`.

If you want to run the tests interactively you can launch the container and enter into it:
```
CONTAINER=$(docker run -d -v <test project location>:/project --env MANUAL=yes mrsheepuk/protractor)
docker exec -ti $CONTAINER sudo -i -u node bash
```
When inside the container you can run the tests at the console by simply invoking `protractor`. When you are done, you terminate the Protractor container with `docker kill $CONTAINER`.

Your protractor.conf.js must specify the no-sandbox option for Chrome to cleanly run inside Docker. A minimal example config would be:

```
exports.config = {
  seleniumAddress: 'http://localhost:4444/wd/hub',
  framework: "jasmine2",
  specs: ['*.spec.js'],
  // Chrome is not allowed to create a SUID sandbox when running inside Docker  
  capabilities: {
    'browserName': 'chrome',
    'chromeOptions': {
      'args': ['no-sandbox']
    }
  }
};
```
