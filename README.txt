The nodeJS script accepts requests and creates a queue for the requests.

The requests are then sequentially sent to another server to be processed.

If the processing server is busy, the requests and kept in queue and the process is repeated until the status of the request is changes to "resolved".