# Rules for ELK SIEM

All rules for ELK SIEM will be stored in coresponding subdirectory, template dir is `00` and each number represents number of subdir from attacks repository.

## Realized attacks

### 01 Jamming detection 
Jamming of the End node detection was combined with the service degradation and we set out to detect and alert and administrator if a given end node was sending less than a given number of messages eg. 2 700 per day.

### 02 Replay attack detection
This attack was unfortunately not successfuly realized, therefore we do not have a detection rule for it.  

### 03 Unathorized gateway movement detection
We created a detection rule to generate an alert if there is a change in the latitude and longtitude syslog messages from the gateways.

### 04 Unathorized database access detection
We created a rule which detect unauthorized access to the reddis database on chirpstack server. 

### 05 MITM attack between application and network server detection
This attack was not possible to carry out therefore we did not attempt to create a detection rule for this.

### 06 Transmission restrictions violation detection
We created a detection rule based on our setup transmission scheme which forced the end devices to send uplink data messages every 30 seconds, so if there are more messages from a given end device within 24 hours we consider it to be transmission restriction violation. 

### 07 Service degradation/anomallies in data values detection
This included anomallies in the meassured value which in our case was a temperature expected to be within the range of 15 to 30 degrees so if the value was outside this given range we considered it an anomally. As for the Service degradation was considered everything which would lead to an end device sending less than 2 700 messages per 24 hour as described in jamming example.

### 08 Bit flipping attack detection
This attack was unfortunately not possible to carry out with our usage of LMIC end node library ergo we did not have to create a detection rule for it. 

