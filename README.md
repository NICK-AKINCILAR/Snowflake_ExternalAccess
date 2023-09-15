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


### Demo 3 : US Address standardization using Smarty.com API REST service.

You will be able to send full of partial US Addresses to Smart.com to get the Standardized USPS version along with additional meta-data like Lat & Long coordinates, Property Type(Commercial, Residential & etc.)

Prerequisites:
1. Register a free account with https://www.smarty.com/docs/cloud/us-street-api
2. Get your API key info (auth-id & auto-token)

~~~ SQL
--  US ADDRESS LOOKUP

CREATE OR REPLACE SECRET streets_api
  TYPE = GENERIC_STRING
  SECRET_STRING = 'auth-id=aba.....773c&auth-token=r8FvKP....IpuI';  --  <YOUR API KEY VALUES HERE>


CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION streets_access_integration
  ALLOWED_NETWORK_RULES = (My_outbounb_network_rule)
  ALLOWED_AUTHENTICATION_SECRETS = (streets_api)
  ENABLED = true;


CREATE OR REPLACE FUNCTION us_address(MyAddress STRING)
RETURNS STRING
LANGUAGE PYTHON
RUNTIME_VERSION = 3.8
HANDLER = 'get_page'
EXTERNAL_ACCESS_INTEGRATIONS = (streets_access_integration)
PACKAGES = ('requests', 'beautifulsoup4')
SECRETS = ('accesskey' = streets_api )
AS
$$
import _snowflake
import requests
from bs4 import BeautifulSoup

session = requests.Session()
mytoken = _snowflake.get_generic_secret_string('accesskey')

def get_page(MyAddress):
  url = f"https://us-street.api.smarty.com/street-address?{mytoken}&license=us-core-cloud&street={MyAddress}"
  response = session.get(url)
  output = response.text.replace("[", "")
  output = output.replace("]", "")
  return output
$$;

select us_address('22 Degroat Road , Sandyston, nj');

create or replace table AddressList
(
    AddressOriginal string,
    AddressRaw variant,
    AddressNew string
);


insert into addresslist values 
('22 Degroat Road , Sandyston, nj', NULL, NULL),
('3331 Erie Avenue, Cincinnati, OH ', NULL, NULL),
('525 South Winchester Boulevard, 95128 ', NULL, NULL);

select * from addresslist;

-- Update the addressList table with full raw Json response packet received for each address.
UPDATE addresslist
SET AddressRaw = parse_json( us_address(AddressOriginal) );

-- Use standard Snowflake SQL Semi-Structured data functions to parse & split the json values in to individual columns.
select 
    addressoriginal,  
    addressraw:delivery_line_1::string || ', ' || addressraw:last_line::string as NewAddress, 
    addressraw:metadata:latitude as lat , 
    addressraw:metadata:longitude as long,
    addressraw:metadata:rdi::string as ZoningType,
    addressraw
from addresslist ;

~~~


### Creating & Registering external functions via Python Snowpark

Following code shows how you can create the identical call_Alexa function from Demo1 using Python & Snowpark. Stage location needs to be a valid internal or external Snowflake stage.

~~~ Python
session.add_packages("requests")

def call_Alexa(topic: str)  -> str:
  import requests
  import _snowflake
  mysession = requests.Session()
  mytoken = _snowflake.get_generic_secret_string('cred')
  url = f"https://api-v2.voicemonkey.io/announcement?token={mytoken}&device=myechodot&text={topic}"
  response = mysession.get(url)
  return response.text



CallAlexa_udf = session.udf.register(call_Alexa, name="call_Alexa", is_permanent=True, replace=True, 
                                      external_access_integrations=["voicemonkey_access_integration"],
                                      secrets={"cred": "deletethis.public.voicemonkey_api"},
                                      stage_location="@DEMO_SNOWPARK.Public.PythonUDF_Stage/PythonUDFS")

~~~

Test your function via Snowpark

~~~ Python
import pandas as pd
  
# initialize list elements
data = ['Snowflake is calling']

df_panda = pd.DataFrame(data, columns=['MESSAGES'])

df_snowpark = session.create_dataframe(df_panda)
df_snowpark.show()

df_snowpark = df_snowpark.group_by(col("MESSAGES")).agg( CallAlexa_udf( col("MESSAGES") ).alias("Result"))

df_snowpark.show()

~~~
