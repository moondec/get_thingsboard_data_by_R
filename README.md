# get_thingsboard_data_by_R
Short R function - reads ThingsBoard data (free edition)

## Some additon libraries are required:
### This package is required for Accessing APIS (HTTP or HTTPS URLS from Web)
library(RCurl)
### This package exposes some additional functions to convert json/text to data frame
library(jsonlite)
### Function body
```{R}
get_thingsboard_data <- function(THINGSBOARD_URL = "https://your.thingsboard.server.io",
                                 x = list(
                                   username = 'YOUR_USERNAME',
                                   password = 'SECRET'), 
                                 DEVICEID="PUT_HERE_DEVICEID", # sensor id
                                 KEYS='Temperature,Humidity,   Press,Precipitation, PM2_5,SolarRad', # set of variables from your sensor
                                 STARTS=round(as.numeric(as.POSIXct("2004-01-01 0:0:0 GMT"))*1000,0), # unix time in milis
                                 ENDTS=round(as.numeric(Sys.time())*1000,0), # unix time in milis (curent time)
                                 verbose=FALSE) {
  url <- paste0(THINGSBOARD_URL,'/api/auth/login')
  #############
  # Get token
  #############
  # headers (same as -H)
  headers <- list('Accept' = 'application/json', 'Content-Type' = 'application/json')
  
  # R structure of the input for the request (same as -d + JSON)
  # x = list(
  #   username = 'YOUR_USERNAME',
  #   password = 'SECRET')
  # url, headers and x are the parameters to be used in R functions to send the request
  # and save the output data in the datajson object
  # postForm is the RCurl function to send the request using the POST method
  # toJSON is the jsonlite function to convert the R structure of the request in JSON input
  # option "auto_unbox = T" remove [] from json.
  
  datajson <- postForm(url, .opts=list(postfields=toJSON(x,auto_unbox = T), httpheader=headers))
  # datajson contains {"token":"[really long string]","refreshToken":"[really long string]"}, we need just token:
  JWT_TOKEN=fromJSON(datajson)$token
  
  ############
  # Get data
  ############
  # see: https://gist.github.com/Engineer-of-Stuff/129badef3e8de319975d474dedbb9d62
  # curl -X GET "${THINGSBOARD_URL}/api/plugins/telemetry/DEVICE/${DEVICEID}/values/timeseries?keys=
  # ${KEYS}&startTs=978307200000&endTs=${ENDTS}&agg=NONE&limit=100000000" 
  # --header 'Accept: */*' --header "Content-Type:application/json" --header "X-Authorization: Bearer ${JWT_TOKEN}" -o $OUTFILE
  
  headers2 <- list('Accept' = '*/*', 'Content-Type' = 'application/json','X-Authorization' = paste('Bearer', JWT_TOKEN))
  my.datajson <- httpGET(paste0(THINGSBOARD_URL,'/api/plugins/telemetry/DEVICE/',DEVICEID,'/values/timeseries?keys=',gsub(" ", "",KEYS),'&startTs=',STARTS,'&endTs=',ENDTS,'&agg=NONE&limit=100000000'),.opts=list(httpheader=headers2,verbose=verbose))
  # save data to list
  my.data <- fromJSON(my.datajson)
  # Due to the data were recorded at the server at a specific time (when they came), the timestamp could be different for each variable (KEY).
  # Therefore better is align time outside the function
  return(my.data)
  # my.data$KEY(data.frame)$ts
  #                             $value
  # to change unix ts to R POSIXct format use below command (ThingsBoaard save ts in milliseconds)
  # my.data$KEY(data.frame)$ts <- as.POSIXct(my.data$KEY(data.frame)$ts/1000, origin="1970-01-01")
}
```
