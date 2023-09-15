# Snowflake External Access Demos


### Create a firewall rule to allow Snowflake to communicate with certain web & IP addresses

~~~ SQL
use role accountadmin;

-- NETWORK rule is part of db schema
CREATE OR REPLACE NETWORK RULE My_outbounb_network_rule
  MODE = EGRESS
  TYPE = HOST_PORT
  VALUE_LIST = ('en.wikipedia.org', 'us-street.api.smarty.com', 'api-v2.voicemonkey.io');
~~~










### Demo 1 : Sending Announcement messages to Alexa

You will be able to send text messages to Alexa to announce via this external function using an intermediate API service from VoiceMonkey.io

Prerequisites:
1. Register a free account with [voicemonkey.io](https://console.voicemonkey.io/)
2. Register your Alexa device to this service & name it "myechodot"
3. Get your API key

~~~ SQL

-- ALEXA DEMO
-- SECRET object is part of a db schema 

CREATE OR REPLACE SECRET voicemonkey_api
  TYPE = GENERIC_STRING
  SECRET_STRING = '365067b36ad7b1a78......ad2570849';  -- < Your API Key here >

  -- CREATE OR REPLACE SECRET oauth_token
  -- TYPE = OAUTH2
  -- API_AUTHENTICATION = google_translate_oauth
  -- OAUTH_REFRESH_TOKEN = 'my-refresh-token';


GRANT READ ON SECRET voicemonkey_api TO ROLE sysadmin;
  

 -- EXTERNAL ACCESS INTEGRATION is a account level object
 
 CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION voicemonkey_access_integration
  ALLOWED_NETWORK_RULES = (My_outbounb_network_rule)
  ALLOWED_AUTHENTICATION_SECRETS = (voicemonkey_api)
  ENABLED = true;

GRANT USAGE ON INTEGRATION voicemonkey_access_integration TO ROLE sysadmin;



CREATE OR REPLACE FUNCTION call_Alexa(topic STRING)
    RETURNS STRING
    LANGUAGE PYTHON
    RUNTIME_VERSION = 3.8
    HANDLER = 'get_page'
    EXTERNAL_ACCESS_INTEGRATIONS = (voicemonkey_access_integration)
    PACKAGES = ('requests')
    SECRETS = ('cred' = voicemonkey_api )
AS
$$
import _snowflake
import requests

session = requests.Session()
def get_page(topic):
  mytoken = _snowflake.get_generic_secret_string('cred')
  url = f"https://api-v2.voicemonkey.io/announcement?token={mytoken}&device=myechodot&text={topic}"
  response = session.get(url)
  return response.text
$$;

-- TEST
SELECT  call_Alexa('This is Snowflake calling to let you know Nicks code works ') ;

~~~



### Demo 2 : We Scraping from Wikipedia.com

You will be able to extract text context from various Wikipedia pages based on the topics of interest. This demo will retreive pure HTML content from a web page the clean up & remove HTML code using beautifulsoup4 Python library to return only the text content of a page. 

~~~ SQL

--  WIKI WEB SCRAPING


CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION wiki_www_access_integration
  ALLOWED_NETWORK_RULES = (My_outbounb_network_rule)
  ENABLED = true;



CREATE OR REPLACE FUNCTION wiki_via_python(topic STRING)
RETURNS STRING
LANGUAGE PYTHON
RUNTIME_VERSION = 3.8
HANDLER = 'get_page'
EXTERNAL_ACCESS_INTEGRATIONS = (wiki_www_access_integration)
PACKAGES = ('requests', 'beautifulsoup4')
--SECRETS = ('cred' = oauth_token )
AS
$$
import _snowflake
import requests
from bs4 import BeautifulSoup

def get_page(topic):
  url = f"https://en.wikipedia.org/wiki/{topic}"
  response = requests.get(url)
  soup = BeautifulSoup(response.text)
  return soup.get_text()
$$;

select wiki_via_python('Snowflake_Inc.');

create or replace table wiki_websites (topic string);

insert into wiki_websites values ('BMW'), ('Snowflake_Inc.'), ('Hard_Rock_Cafe');

select * from wiki_websites;

select topic, wiki_via_python(topic) from wiki_websites;
~~~

