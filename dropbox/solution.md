# Dropbox 

## Requirements
Core Requirements
- User should be able to upload a file
- User should be able to download a file
- System should sync the files across the devices
- User should be able to share files with other users
- User should be able to view files shared with him

## Non functional requirements
Core Requirements
- The system should be highly available (prioritizing availability over consistency).
- The system should support files as large as 50GB.
- The system should be secure and reliable. We should be able to recover files if they are lost or corrupted.
- The system should make upload, download, and sync times as fast as possible (low latency).

## Entities

- User
- File
- User File mapping
- File ACL
- User file shared mapping (File ACL aggregated view from recipient users perspective)
- File events - file created, file deleted, file shared with me

## APIs
- GET /file - list all files user has
- GET /file/{fileId} - get file by id
- POST /file - upload a file
- GET /file/events - get all the file events after a certain timestamp, paginated
- DELETE /file/{fileId} - delete a file

## Observations and design highlights
1. Essentially use can view 2 kinds of files, his own files and the ones shared with him.
2. Files would be persisted in the object storage
3. It would be best to chunk up the files and upload them parallely from the client machine
4. We'll store a file chunk mapping in MongoDB
5. We'll also store the user files mapping in MongoDB
6. We'll also keep a table for the file share and access policy in MongoDB indexed on the recipient user id, to look up shared files with curr user
7. Clients would keep doing long polling on the backend to look for file events like new files shared with user, new files uploaded, files downloaded, file deletes
8. We can use a timestamp, which client device can use to fetch the file events after a certain timestamp
9. Client will pick some events, process them ie either drop the file, pull file, then keep pulling for newer updates
10. Client devices would be eventually consistent with the actual file state
11. Clients will keep the local state in a sqlite db to keep track of the events consumed, current state and where to start consuming events from

## Datastore
1. Object store for files
2. MongoDB for user files mapping - partitioned on user id
3. MongoDB for File ACL - partitoned on file id
3. MongoDB for user shared files mapping - partiioned on recipient user id, this is CDC view of file ACL from recipient users perspective, eventually consistent
4. MongoDB for File events - partitioned on recipient user id (for own files, user itself is recipient, for share the recipient will be diff)

## Architecture diagram

![Architecture diagram](./assets/dropbox.drawio.svg "Architecure diagram")

## Components
1. Storage service
2. File service
3. Event workers

## Flows

1. File upload
    - User logs in
    - User hits the upload button for a file
    - Client device breaks down the files into chunks
    - Client device registers the file with the file service, which returns the chunks persigned object url
    - Client device uploads the chunks to the objectstore
    - Client device registers the chunk upload with chunk hash, sys ack
    - Once all the chunk upload is done
    - Client registers the file upload, system registers the upload, adds the hash of the file and ack
    - CDC will capture the file upload
    - CDC will insert a file uploaded event into the event store
2. File sync
    - Client does long poll for events
    - Client gets an event for file upload
    - Client gets the file id from event
    - Client gets the chunk details from storage layer
    - Client gets the chunks and assemble the file on device
    - Client keeps looking out for further events
3. File share
    - User shares a file with another user
    - System inserts this entry into ACL table
    - CDC will pick up this entry
    - CDC will insert a entry in user shared file for the recipient user
    - CDC will add an event of file shared which recipient user can read

