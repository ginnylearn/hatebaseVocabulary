# hatebaseVocabulary
Use R to retrieve personal token and get vocabulary from Hatebase.org

# Load packages

```{r message=FALSE, warning=FALSE}
# Load packages
library("httr") # API calls
library("jsonlite") # Extracting content from Hatebase repo
library("tidyverse") # Several data processing packages
library("data.table") # Convert list to data frame
```

# How to extract information from a response object

Before getting started, here's a quick breakdown of how these functions work. In both phases, you'll use `POST()` to retrieve a response ojbect (`r`), and then you'll use the `content()` and `fromJSON` functions to extract information from the response object. 

## POST()
The `POST()` function from the `httr` package lets you provide your personal information to Hatebase, and it provides you with informatin too. In the Authentication phase, it'll return your `token`, and in the Query phase, it'll ruturn the total number of `nPages`.

*`POST()` arguments:* 
* url: uses `paste0()` so that the version can be updated
* body: a `list` that contains arguments that vary based on the type of request (e.g. api_key, token, format, etc.)  
* encode: you'll use this in the Authentication phase since you're posting information in a form

## content() & fromJSON()
In order to extract information from your response object (`r`), you'll need to take a couple more steps: 
1. Use the `content()` function from `httr` to extract the content from `r` and save it as a new object 
2. The `fromJSON()` function from the `jsonlite` package allows you to quickly parse through your new object object. In addition, you can use R's subsetting features to access specific data points.  

# Authentication phase: Retrieving token 
The `hatebaseToken` function uses your API key to retrieve your personal token from your Hatebase account. 

You'll need two things to get started: 

1. Version number (`version`): current version can be obtained from the [Hatebase github page](https://github.com/hatebase/Hatebase-API-Docs/tree/master/current) 
2. API key (`api_key`): obtained from your [Hatebase account](https://hatebase.org/)

```{r}
# Function args
version = "4-3" # current version as of Aug. 2019
api_key = "yourAPIkey"

# Create retrieve token function
hatebaseToken <- function(version = NULL, api_key = NULL) { 
  
  # Handshake 
  r <- POST(url = paste0("https://api.hatebase.org/", version, "/authenticate"), 
            body = list(api_key = api_key), 
            encode = "form")
  
  # Retrieve token
  rText <- content(r, as = "text") # extract content 
  token <- fromJSON(rText)$result['token'] %>% # parse text & subset object
    as.character() # character format
  
  return(token)
}

# Run retrieve token function 
token <- hatebaseToken(version = version, api_key = api_key)
```

# Query phase: Get vocabulary 

In order to retrieve the Hatebase vocabulary, you'll need three things:

1. Version number (`version`): already created in Authentication phase
2. Token (`token`): retrieved in Authentication phase
3. Total number of pages in the Hatebase vocabulary dataset (`nPages`): you'll need to retrieve this before you can download the dataset since the total number of pages changes with every new update

## Number of pages 

In order to retrieve the number of pages in the Hatebase vocabulary dataset, you need to make a vocabulary query without specifying the page numbers. The response object (`r`) contains the number of pages, but you'll need to do a little bit of data wrangling to extract the information (for a description of these steps, refer to **How to extract information from a response object**) 

```{r}
# Retrieve page numbers 
hatebaseVocabPages <- function(version = version, token = token) {
  
  # Get vocabulary information 
  r <- POST(url = paste0("https://api.hatebase.org/", version, "/get_vocabulary"), 
            body = list(token = token, format = "json"))
  
  # Total number of pages
  rText <- content(r, "text")
  nPages <- fromJSON(rText)$number_of_pages
  
  return(nPages)
  }

nPages <- hatebaseVocabPages(version = version, token = token)
```

## Get vocabulary from every page
Now that you've got the `version`, your `token`, and the total number of `nPages`, you can use a for loop to iterate your vocabulary query across every page in the Hatebase dataset. 

The `inputParams` is a list of required paramaters, and you can add additional parameters to limit your request specific languages, types of words, etc. 

For a description of all available input parameters, see the [Hatebase github page](https://github.com/hatebase/Hatebase-API-Docs/blob/master/current/v4-3/get_vocabulary.md)

```{r}
# Initialize list
hateList <- vector("list", nPages)

# Input parameters
inputParams <- list(token = token, 
               page = "i", 
               format = "json")

# For loop
for(i in 1:nPages){ 
  hateList[[i]] <- POST(url = paste0("https://api.hatebase.org/", version, "/get_vocabulary"), 
                        body = inputParams) %>%
    httr::content() %>%
    fromJSON() 
  }

```

Once you've retrieved the Hatebase vocabulary, you can convert it from a list to a data frame by taking the following steps: 
```{r}
# Convert to data frame
hateDF <- lapply(hateList, '[', 'result') %>%
  map(as.data.frame) %>% 
  data.table::rbindlist()

# Save as csv
write_csv(hateDF, "hateBase.csv")
```
