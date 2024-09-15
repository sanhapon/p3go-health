## How to run scraper
```
/$PATH/spiders % python3 runner.py
```

## Here is the scaper code:
```
# p3go_spider.py

from pathlib import Path
import scrapy


class P3goSpider(scrapy.Spider):
    name = "p3go"
    def __init__(self, url=None, *args, **kwargs):
        super(P3goSpider, self).__init__(*args, **kwargs)
        self.start_urls = [url] if url else []


    def parse(self, response):

        sec = response.css('main>section').get()
        yield {
            'sec': sec    
        }

```

```
#runner.py

import subprocess
# subprocess.check_call(['pip', 'install', '-U', '-q', 'google-generativeai'])

from p3go_spider import P3goSpider
from datetime import date
import json
from scrapy.crawler import CrawlerProcess
from scrapy import signals
import google.generativeai as genai

genai.configure(api_key="AIzaSyCUed-bWeU2uzVlhpWBkWCQ7DjGm9w-pbE")

model = genai.GenerativeModel(
    "models/gemini-1.5-flash",
    system_instruction="You are a writer expert that specializes in english to thai translator, please return the markdown code. Do not give an explanation for this code."
)

def get_content_from_llm(messages):
    response = model.generate_content(
                messages,
                generation_config=genai.types.GenerationConfig(
                            candidate_count=1,
                            temperature=1.0)
            )
    return response.candidates[0].content.parts[0].text
    

def item_scraped_callback(item, response, spider):
    scraped_content = item['sec'] 
    messages= f"""Write thai blog from the following html snippet which is enclosed with triple quote.'''{scraped_content}'''""" 
    content = get_content_from_llm(messages)
    
    messages= f"""Write blog title and description, seo keywords and slug url in thai language for thai blog from the content which in enclosed with tripple qoute. Response in json format using title, description, keywords and slug as json property.
        Important: 1. Write interesting title, concise and differ from any text in the content. 2. Write slag url in thai lanaguge.
        '''{content}'''
    """ 
    header = get_content_from_llm(messages)
    header_corrected = header.replace("`", "").replace("json", "")
    header_dict = json.loads(header_corrected)

    content_corrected = content.replace("`", "").replace("markdown", "")
    write_blog(header_dict['title'], header_dict['description'], header_dict['keywords'], header_dict['slug'], content_corrected)

def write_blog(title, description, keywords, slug, content):
    today = date.today()
    formatted_date = today.strftime('%Y-%m-%d')

    blog = f"""---
title: '{title}'
description: '{description}'
pubDate: '{formatted_date}'
keywords: '{keywords}'
heroImage: '/blog-placeholder-3.jpg'
---

{content}

"""

    with open(f"./content/{slug}.md", 'w') as file:
        file.write(blog)
        

process = CrawlerProcess()
crawler = process.create_crawler(P3goSpider)
crawler.signals.connect(item_scraped_callback, signal=signals.item_scraped)
# process.crawl(crawler)
process.crawl(crawler, url='https://www.eatright.org/health/essential-nutrients/supplements/supplements-for-breastfed-babies')
process.start() 

```
