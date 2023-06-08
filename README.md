import snscrape.modules.twitter as sntwitter
import pandas as pd
from pymongo import MongoClient
import streamlit as st
import datetime
import base64
import json


def scrape_tweets(keyword, start_date, end_date, tweet_limit):
    
    start_datetime = datetime.datetime.combine(start_date, datetime.datetime.min.time())
    end_datetime = datetime.datetime.combine(end_date, datetime.datetime.max.time())

    
    start_datetime = start_datetime.replace(tzinfo=None)
    end_datetime = end_datetime.replace(tzinfo=None)

    tweets = []
    for tweet in sntwitter.TwitterSearchScraper(keyword).get_items():
        
        tweet_date = tweet.date.replace(tzinfo=None)
        if tweet_date < start_datetime or tweet_date > end_datetime:
            continue

        tweets.append({
            'date': tweet.date.strftime("%Y-%m-%d %H:%M:%S"),
            'id': tweet.id,
            'url': tweet.url,
            'content': tweet.rawContent,
            'user': tweet.user.username,
            'reply_count': tweet.replyCount,
            'retweet_count': tweet.retweetCount,
            'language': tweet.lang,
            'source': tweet.source,
            'like_count': tweet.likeCount
        })

        if len(tweets) >= tweet_limit:
            break

    return tweets



def upload_to_mongodb(data):

    connection_uri = "mongodb://localhost:27017"

    
    client = MongoClient(connection_uri)
    db = client['Twitty']
    
    collection = db['tweets']

    
    result = collection.insert_many(data)



def download_as_csv(data):
    df = pd.DataFrame(data)
    df.to_csv('tweets.csv', header=False, index=False)



def download_as_json(data):
    json_data = json.dumps(data, indent=4)
    b64 = base64.b64encode(json_data.encode()).decode()
    href = f'<a href="data:file/txt;base64,{b64}" download="tweets.json">Download JSON File</a>'
    st.markdown(href, unsafe_allow_html=True)



def main():
    
    st.title("Twitter Scraping App")
    keyword = st.text_input("Enter a keyword or hashtag:")
    start_date = st.date_input("Select the start date:")
    end_date = st.date_input("Select the end date:")
    tweet_limit = st.number_input("Enter the number of tweets to scrape:", min_value=1)

    
    if st.button("Scrape"):
        
        scraped_tweets = scrape_tweets(keyword, start_date, end_date, tweet_limit)
        st.table(pd.DataFrame(scraped_tweets))

    
    if st.button("Upload to MongoDB"):
        upload_to_mongodb(scraped_tweets)

    
    if st.button("Download as CSV"):
        download_as_csv(scraped_tweets)

    
    if st.button("Download as JSON"):
        download_as_json(scraped_tweets)



if __name__ == "__main__":
    main()









