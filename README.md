
# CSV Image data

 A system to efficiently process image data from CSV files

 


####  Tech stack
- NodeJS: Main server
- redis: to implement queue
- docker: to run multiple service as container
- firebase: to store output csv file and images
- nginx: reverse proxy [open only 1 port and redirect to multiple service]
- mongodb: database [NoSQL is faster than sql]
## Public Routes
### /csv/upload
POST
```
type:formData
key: file
```
response
```
{
    "requestId": "382c81e1a71a32d8eb9595cc5c4a4112"
}
```
### /csv/status
GET
```
/status?requestId
```
response

```
{
    "status": "csvgenerated",
    "csv": "https://storage.googleapis.com/csvimg.appspot.com/db66d46b7a4e6ff9c15bb274577fac84_output.csv"
}
```

```
{
    "status": "pending" or "processing" or "completed"  or "failed"
}
```
- pending: image being waiting in queue 
- processing: request is being processing by imageprocessing service
- completed: processing (ouput url)have been completed but csv not generated
- csvgenerated: output csv file generated

## Private Route
### /csv/hook
POST
```
{requestId}
```
response 

```{status:"ok"}```
## services
## 1.csv
 - accept csv file upload
 - check if csv file correctly formated
     - headers 'serialNumber', 'productName', 'inputImageUrls' exist
     - all imageurls have valid format
- check status by requestId
- generate output csv file when imageprocess service invoke "/hook"

## 2. redis queue
  all request are pushed in the queue they kept waiting for their turn to be processed

## 3. imageprocess
- when the queue is not empty, it take a request from queue 
- process all image urls one by one
- compress to 50%
- upload to firebase storage and get output url
- invoke csv service by private url "/hook" when all  produdct have been processed for the given requestId

### database schema

```


const imageSchema = new mongoose.Schema({
  serialNumber: { type: String, required: true },
  productName: { type: String, required: true },
  inputImageUrls: { type: [String], required: true },
  outputImageUrls: { type: [String], default: [] }
});

const requestSchema = new mongoose.Schema({
  requestId: { type: String, required: true, unique: true },
  status: { type: String, required: true, enum: ['pending', 'processing', 'completed','csvgenerated', 'failed'], default: 'pending' },
  csvOutput:{type:String},
  images: [imageSchema]
}, {
  timestamps: true
});

```

## error
```
{
    error:"server error"
}
```
```
{
    error:"Request not found"
}
```
## Sample input file


| serialNumber | productName | imageUrls
|--------------|-------------|---------
| 1            | book        |https://www.public-image-url1.jpg,https://www.public-image-url2.jpg

## Sample output file

| serialNumber | productName | inputImageUrls | outputImageUrls |
|---------|---------|---------|---------|
| 1   | book  | https://www.public-image-url1.jpg ,https://www.invalid-url2.jpg   | https://storage.googleapis.com/csvimg.appspot.com/ddafee145049c2db.jpg,failed_to_fetch   |

### Note
if a url is failed to download it will show 
```failed_to_fetch```

## workflow
### csv service
- accept csv file 
- validate csv file 
- parse comma separated urls
- update status as pending 
- save to mongodb
- push request to queue
- send requestId as response

### image service

- pick request from queue
- update request status  as processing in database 
- for each product do the followings
   - download image for each url
   - reduce to 50%
   - save to firebase
   - store the urls in array outputImageUrls
- update the request object and marking status as "completed"
- invoke the "/csv/hook" passing {requestId}

### csv service
- generate output csv file from request object
- save to firebase
- get public url
- update database marking request object csvOutput=url and status=csvgenerated
        
## RUN locally
 - install nodejs,docker,docker-compose
 - clone the project
 - goto inside folder
 - run the script.sh
 - run docker-compose build
 - run docker-compose up
### Note
 you may need to use "sudo"

## postman collection
https://api.postman.com/collections/16080178-2c71c954-dca6-4886-844c-c31a8ae84573?access_key=PMAT-01J2BVJAR68KCGZ6TK911TK2MS

## design
[![client-2.png](https://i.postimg.cc/GpkgC50Y/client-2.png)](https://postimg.cc/cKLB7TL4)
