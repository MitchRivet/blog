+++
title="Analyzing Reddit Comments: Scraping (Part 1)"
date = "2016-11-06T22:17:00+02:00"
tags=["tutorial", "beginner", "NodeJs", "ExpressJs", "data-science"]
+++

A place to start when getting into data science is to find data sets that are personally relevant and/or interesting. one place i like to get data to understand more about internet discourse is reddit.

# How to scrape reddit

if we want to download data from reddit, we essentially have two options. one is making an oauth request which includes logging into the reddit api with a reddit account. the downside to this method is that each user is limited to 60 oauth requests per hour, which makes it hard to download an entire comment thread and all of the relevant user meta data from comments in the thread (popular discussions usually break over 1,000 comments).

another option is to scrape reddit, or in other words use a server to visit web pages and actually download the html content we're looking for ourselves, which we can then parse into whatever relevant bits we are looking for. reddit makes this process even one step easier by exposing most of their endpoints as json strings. we can see this by going to any thread on reddit and attaching a '/.json' to the end of the url. this allows us to easily convert these giant strings straight to json without having to create our own structure. the downside of this method is that we sometimes end up with incomplete data sets. consider the links on reddit which say 'more comments' -> by scraping reddit our data will include this 'more comments' link without us being able to access that data directly. we could tell our server to collapse all those links, but lets just start with a general data set to get going.

lets use node and expressjs to do this. we start by npm installing and setting up a basic expressjs app. expressjs is a routing framework for nodejs, a javascript development environment for creating servers.

<pre><code class="javascript">

    //declare our initial dependencies
    var express = require('express');
    var request = require('request');

    //initialize express
    var app = express();

    app.get('/scrape', (req, res) => {

            res.json({message: 'this is your endpoint'});
    });

    app.listen('8081');     

    console.log('Listening on port 8081');
    exports = module.exports = app;

</pre></code>


lets save this file as 'reddit_request.js'. we can now fire up our server by opening the command line into the directory where we saved this file and running:

<pre><code>node reddit_request.js</code></pre>

Thanks to our console.log, we should get a message printed in the terminal saying 'Listening on port 8081'. If everything is working correctly, we should be able to open our web browser to localhost:8081/scrape and see the json we returned fromt that endpoint.

if that's all working, we have just created a server with an endpoint called '/scrape'. when use express to do our routing, we can use our client to make a get request to the server.

## Making Requests

Now lets use the request module to download the content we want from Reddit. Request is a built in NodeJS module and can be used to make HTTP and HTTPS requests, allowing us to download html or in this case, JSON strings. We are going to modify our endpoint like this:

<pre><code class="javascript">

    //
    //

    app.get('/scrape', (req, res) => {
        var url = 'https://www.reddit.com/r/politics/comments/5bu1u6/if_you_vote_for_trump_you_own_the_racism_yes_you/.json';

        //note we won't be using the html return value in the callback, but I included it so you would know it's there
        request('url', (err, response, html) => {
            if (err) {
                console.log(err);
            } else {
                let commentJSON = JSON.parse(response.body);
                res.json(commentJSON);
            }
            });
    });

    //
    //

</code></pre>

Now, reload the server with the 'node reddit_request.js' command, make the same request to localhost:8081/scrape in your client. you should see a giant mess of json that is a replica of whatever thread you were looking at. I recommend using chrome and a chrome module to make the JSON collapsable and much more readable.

## Parsing Comments

Notice that the structure of the comments is nested in the same way they are visible on the html page. what if we want to analyze each comment individually? we would want to place each comment into a flat array. if we want to get a structure like this we will need to use recursion. We essentially want the computer to go through each comment and ask "does this comment have any replies? if yes, then i will run the same operation of the array of comments inside of the reply property. if not, then i will continue processing the top level of the comments array".

For this task, we can use a new syntax to es6 called generators. Add the following function globally, above your '/scrape' endpoint. We will still need to call the function later on, but lets first define and understand it.

<pre><code class="javascript">
    //
    //

    //we're passing in all of our returned comments
    function *findReplies(comments) {
        if (!comment) {return ;}

        //looping through all of the comments
        for (var i = 0; i < comments.length; i++) {

            //which ever step of the array we are on, grab the data
            var commentMeta = comments[i].data;

            //checking that the comment was not deleted    
            if (commentMeta.body && commentMeta.author !== "[deleted]") {

              allWordsString = allWordsString + commentMeta.body + '';
              var score = sentiment(commentMeta.body);

              yield {
                author: commentMeta.author,
                body: commentMeta.body,
                score: score
              };
            }

            if (commentMeta.replies) {
              yield *findReplies(commentMeta.replies.data.children);
            }
          }
    }

    app.get('/scrape', (req, res) => {

    //
    //
</code></pre>  

Every time we hit a yield statement in this function, whatever we yield will be part of a larger return. This happens in two places: the first when we grab the comment meta data we are looking for, and the second when we call the generator itself if the comment contains comments within. A function calling itself is [Recursion](https://en.wikipedia.org/wiki/Recursion). This syntax yields itself nicely to what we are trying to do.

Now, how do we call this function?

<pre><code class="javascript">
//
//

app.get('/scrape', (req, res) => {

    var allComments = [];

    var url = 'https://www.reddit.com/r/politics/comments/5bu1u6/if_you_vote_for_trump_you_own_the_racism_yes_you/.json';

    request('url', (err, response, html) => {
        if (err) {
            console.log(err);
        } else {
            let commentJSON = JSON.parse(response.body);
            let header = response.body[0];
            let comments = commentJSON[1].data.children;

            var getReply = findReplies(comments);
            var replyGrab = getReply.next();

            //the generator has a state that we can manage inside of a while statement
            while (!replyGrab.done) {
              allComments.push(replyGrab.value);
              replyGrab = getReply.next();
            }

            //once the generator is finished, we can send our parsed data back to the client
            if (replyGrab.done) {
                res.json({header: header, comments: allComments });
            }
        }
        });
});
//
//
</code></pre>

Now, open your browser again and double check that the data is being sent back.

## Asynchronously fetching user data

coming soon

## Writing data to JSON File

coming soon
