Is HackerNews a good predictor of startup success?
Sentiment Analysis on HackerNews
Introduction
Hacker News is a social news website focusing on computer science and entrepreneurship. It is run by Paul Graham’s investment fund and startup incubator, Y Combinator. In general, content that can be submitted is defined as “anything that gratifies one’s intellectual curiosity”. Wikipedia
Hacker News
Edit descriptionnews.ycombinator.com

It’s redit for technology in a nutshell. we will be focusing on the posting in the “show” portion of the site where users are able to showcase a very early version of their work to community to get feedback. Notable showcases of projects before they took off are DropBox : https://news.ycombinator.com/item?id=8863 and Gitlab : https://news.ycombinator.com/item?id=4428278
“anything that gratifies one's intellectual curiosity”
The whole purpose of showing your early work on Hacker News is to get an opinion about your idea and get a realistic picture on usability and/or feasibility. users get a simple user interface (much like other social network sites) to vote up/down or start a discussion by adding comments and engaging with the poster or other members of the community. 

Our goal 
Being one of the platforms with a vast collection of tech savvy, opinionated and talented users Hacker news has proven to be a place where ideas get momentum and the creator gets a valuable insight into the viability of what is showcased. As part of our project, we would like to analyse the sentiment of the discussions/comments for the story to find if the discussions were reflective of the future success/failure of the company. 
Dataset
Hacker News data including the stories and comments are available in google cloud BigQuery. The table called “full” contained all the stories and comments have made a data-set containing all stories and comments from it’s launch (2006) available for public.Each story contains id, author who made the post, when it was posted and the number of points story received and so forth. this data is made available by google on the cloud and we were able to write big query SQL to probe the slice of the data we wanted. 

All the data we are looking for lives in the “full” table and the schema of that table looks like 


We used the type column to determine if a specific record is a story of a comment. We were unable to utilize the table comment and story because they were stand alone tables with no column to relate them. Below table describes the counts of each type of entry in the “full” table.

Post Type
Count
comment
18,184,954
story
3,517,469
pollopt
12,100
job
12,562
poll
1,779

For this analysis we decided to use a subset of the data available, specifically 2017. This will allow the posted project to cure. Meaning there would be enough time for active comments to die down and give us more information outside of hacker news to evaluate the success of the project. On top of that a smaller number of projects means reduced number of computation hours on the cloud.
We utilized numerous bigquery sql statements to slice and dice the dataset to get further insights. It was relatively easy to write the queries if you are familiar with SQL syntax.  For example we used SQL aggregate query to pull the counts in the table above:

SELECT type,COUNT(*) postcount
FROM `bigquery-public-data.hacker_news.full`
GROUP BY type
Intuitively we figured having more than just one or two comments on a project would be the bare minimum for us to do sentiment analysis and make deductions based comments. By probing into the data we found that the majority of the projects posted actually receive less that 5 comments and most receive not comments at all. As shown in the chart below.

Comments of comments
One of the challenges we faced with the data structure of the schema is the parent child relationship that does not really go well with SQL. in order to get the full conversation that pertains to a story, we would need to recursively look through the table to find all the children comments. Our initial analysis showed that most of the stories do have comments that go beyond the top level, where users and the original poster will have a back and forth conversation.

It is expected for social network sites to have discussions, flames and feedback chains that go beyond the true content of the original submission. And as it can be observed from our analysis above most of the stories follow up comments. Only 2,194 (~20% of the projects) have only top level comments and close to 40% of the projects have at least 75% of the comments are top level. We did expect this from the data but our initial hunch to disregard these follow up comments is valid when we look at the sentiment analysis in the child comments in the coming sections. 
It is worth mentioning here that we found it quite challenging to come up with a bigquery SQL statement that can do recursion. Our attempt on using CTE (common table expressions from MS SQL) failed since bigquery doesn’t support it. We also hit a wall when we blatantly tried to perform multiple self joins (100+), as the platform doesn’t allow queries that are that complex. We used the query below to pull comments 10 levels deep, so that we can further analyse the impact of child comments.


Community Comment Distribution
We were also interested to see if most of the comments are coming from the same set of individuals. By pulling the top 200 commenters we found out that there are indeed frequent commenters but it doesn’t look like the platform is dominated by few users. Later on we will attempt to see if these users are objective or not (that is if they tend to be always negative or positive)

When users post their projects on hacker news, they normally associate it with a URL where community users go, to review their work. This is where we would go to assess if the project is still active and going. After looking at the data we were working with, we saw that a vast majority of the projects are actually hosted in github (as shown below). 


Data Pre-processing 
Computing startup success
One of the core questions we are trying to answer is if a project posted on hacker news has been successful or not based on feedback from the community. But that particular information is not readily available in the dataset. And requires some effort on our part to go out to the www and make that determination to the best of our abilities. 
At first we would like to check whether their url is still active till now. However, we found out that most of the project is hosted by github.com which we expected it will active for sure. So the idea to check the url directly is not good enough.
For most most of the url is github urls, we could check the activity of the github projects. Since it does not make sense to check the activity just after one story is posted. We decided to check whether the projects get updated after the story is posted 6 months later. If the project gets any activity, we consider it is successful.
For the project is not hosted by github, we find alexa.com could provide some public information for a website. It support API access. We use the rank of the website to decide whether it is active or not. If the rank is not null, we consider it is successful.
Concerns and discussion:
We use different methods to decide the successfulness of a project. The github method is good enough for a startup project in the year of story is posted. The alexa.com provide the information of the project till now. The latter method is much stronger than the first method. However, we do not think we need the activity till now to indicate the successfulness of a startup projects. If the project is interesting, the first year is good enough. For another reason, the non-github hosted projects is a small portion of the whole dataset, we think the much stronger method do not affect the model a lot.
There maybe another method to indicate the successfulness is to check the number of activities for github hosted projects. And checking the rank of the non-github hosted projects. In this way, it could provide fine granularity of the successfulness. The main problem is how to normalize the data between two methods. 
To conclude the discussion, we decided to use the binary tag for our dataset.
 Sentiment Analysis 
What is Sentiment Analysis?
Sentiment analysis is the automated process of analyzing text data and classifying opinions as negative, positive or neutral. 
Usually, besides identifying the opinion, these systems extract attributes of the expression
Polarity: if the speaker express a positive or negative opinion,
Subject: the thing that is being talked about,
Opinion holder: the person, or entity that expresses the opinion.
There are many off the shelf libraries/tools out there that are easy to use and get you up and running with minimal effort. They have varying specialization where they focus on some aspect of sentiment analysis. The focus might be a fine grained opinion that is more than just positive,neutral and negative or it might focus on the emotions associated with this sentiment. Like if the content projects a happy, angry or sad feeling. choice of tools normally rests on what is important for a specific use case. 
How do sentiment analysis algorithms work?
Before proceeding with our goal we thought it would be nice if detail a very high level intro into how these algorithms work.There are many approaches that are part of the NLP field of study, used in doing sentiment analysis. The simplest method that is easy to understand is the rule based approach.
Rule Based : with this approach we keep a list of positive and negative tokens (words) and just do a count to compare the appearance of positive words against the negative words to decide if the statement is more positive or more negative. 
The other approach that is less restrictive is using machine learning. Sentiment analysis is modeled as a classification machine learning problem. And hence the model is fed with text content and it is used to come up with a category. The visual below from https://monkeylearn.com/sentiment-analysis/ showcases this approach nicely.

Mostly libraries have better success by combining these techniques (hybrid).
VADER
Our main focus when choosing a tool to do sentiment analysis on Hackernews comments was ease of use and simpler result for further analysis. So we decided to go with primary with VADER and played with Textblob to compare the tools.
VADER (Valence Aware Dictionary and sEntiment Reasoner) : is a lexicon and rule-based sentiment analysis tool that is specifically attuned to sentiments expressed in social media
One of the good things that stood out from this library was the fact that the api not only returns positive, negative and neutral but it also return by how much? it is particularly useful in our case since we wanted to evaluate how the positiveness or negativeness impacted the success of a project. It is also worth pointing out that it is fast.
VADER returns four values for each sentence it analyzes, 
Positive, negative and neutral represents the portion of the sentence that is positive, negative and neutral respectively. So naturally they all should add up to 1. Below is an example of these scores.
Show a table of data from the comments with interesting data
Setting up the code and getting sentiment analysis is quite simple using VADER
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
analyser = SentimentIntensityAnalyzer()
TextBlob
TextBlob is a simple python library for performing NLP tasks. It provides a simple API to do tasks like part of speech tagging, noun phrase extraction, classification and sentiment analysis among other things. We incorporated this library into analysis to add more features the dataset and do some comparison with the other tool we are using and see if the results are any closer.
For sentiment analysis TextBlob returns two values. 
Polarity : ranging between -1 and 1, this value indicates if the text is positive (> 0) or negative (<0)
Subjectivity: ranging between 0 and 1, this value indicate if the text contains information that factual (closer to 0) or more of opinion (closer to 1).
Data analysis
VADER vs Textblob 
When analysing the different scores we got by running the comment texts we observed that the proportion of the polarity of comments in both are not different, where the majority of the comments are positive and similar amounts are negative. But we were surprised by the scatter plot below that shows how a comment is interpreted by both. we were expecting a cluster that sits close to a diagonal showing minimal difference between the scores from both. But the result did not have a pattern. Although we are not trying to establish a relationship here, but it was a good observation.



Feature Engineering
Once we went through scoring comments tied to these stories, we did some feature engineering. Each story is tied to multiple comments, so we would have to find a way to aggregate the sentiment score of each comment to come up with a single value. We attempted a couple of approaches to come up with a score that is useful to the model and descriptive of the comments attached to the story.
Averaging the score
Our first intuition was to use the average, maximum and minimum scores of comments under a story and see if they have any impact on the outcome of the project. In this aggregation we considered the distinction between only using the top level comments vs using children comments.
Subjectivity 
One of the scores from Textblob is number that tells how subjective a comment is. Using this value we added filters to on our aggregation to account for only the comments that subjective and another feature that accounts for comments that are more objective. This approach helps describing how objective a comment is. And we thought it would be interesting to see if objectivity or subjectivity supplements judgement. From the data we were able to see that the positive comments seem to be more subjective and negative comments tend to be more objective. So we thought it would be interesting to add that as a feature to our model.

Top level Comment ratio
We made an effort to separately represent the sentiment scores for comments that are on the top level (where the community added a comment directly to the story) and children comments. Intuitively we believed that the top level comments are the most relevant comments on the story. In order for us to account for the child comments we thought it would be useful to have a high level value (in this case the ratio of the top level comment against the total number of descendants) as a feature.

Result

Feature importance

Prediction accuracy based on train/test split data for 2017
Conclusion

References

Future Work

