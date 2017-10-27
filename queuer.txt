const express = require('express');
const http = require('http');
const bodyParser = require('body-parser');
const fs = require('fs');
const app = express();
const request = require('request');
const jsonfile = require('jsonfile');
const chokidar = require('chokidar');
const toobusy = require('toobusy-js');
const schedule = require('node-schedule');
const rule = new schedule.RecurrenceRule();
const util = require('util');
const exec = util.promisify(require('child_process').exec);

app.use(bodyParser.json());
app.use(function(req, res, next) {
    res.header("Access-Control-Allow-Origin", "*");
    res.header("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
    next();
});

// Asynchronous move function
async function mv(dir, name) {
  var targ_dir = './req_completed';
  const { stdout, stderr } = await exec('mv '+ dir + '/' + name + ' ' + targ_dir + '/' + name);
  console.log('stdout:', stdout);
  console.log('stderr:', stderr);
}


// Create a timed event here
rule.second = 1;
var jj = schedule.scheduleJob(rule, function() {
        // Get a list of all of the files in the directory (./requests)
        const dir = './requests';
        fs.readdir(dir, (err,files) => {
            if (err == null) {
                if (files.length > 0) {
                    console.log('No of requests queued up: ' + files.length.toString());
                    console.log(files);
                    console.log('Sending forth ... ' + files[0]);

                    // If the size of the list/array is greater than 1, read the data and send to the server
                    var location = './' + dir + '/' + files[0];
                    var filename = files[0]
                    jsonfile.readFile(location, function(err,obj) {
                        if (err == null) {
                            const req = JSON.stringify(obj);
                            const reqObj = JSON.parse(req);
                            console.log('xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx');
                            console.log('Sending request '+ reqObj +' to server');
                            console.log('Sending request '+ req +' to server');
                            console.log('xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx');
                            //mv(dir, filename);

                            // Send the request to the TECH server
                            var headersOpt = {
                                "content-type": "application/json",
                            };

                            request({
                                method:'POST',
                                url:'http://158.69.116.64.xip.io:8080/data',
                                body: reqObj,
                                headers: headersOpt,
                                json:true,
                                },  function (error, response, body) {
                                    if (error == null) {
                                        //mv(dir, filename);
                                        console.log('Request sent and case status changed to completed!');
                                    } else {
                                        console.error(error);
                                        console.log('Try, Try, Try again!')
                                    };
                            //Print the Response
                            console.log(body);
                            console.log(error);
                            });


                        } else {
                            console.error(err);
                            console.error('Could not read json data from file :(');
                        };
                    });

                } else {
                    console.error('Directory (./requests) is empty. No requests in queue!');
                };

            } else {
                console.error(err);
            };
        });
});

//jj.reschedule();

// Ping the server for testing purposes
app.get('/data', function (req, res) {
    const ping = require('node-http-ping');
    //ping('xxx.xx.xxx.xx.xip.io', 8080)
    //  .then(time => console.log(`Response time: ${time}ms`))
    //  .catch(error => console.log(`Failed to ping: ${error}`));

    var body = req.body;
    var file = body["name"]["crowdshot_id"];
    var status = body["name"]["status"];
    var dir = "./requests";
    var filename = file;

    if (status == "resolved") {
    // Call the mv function to move the file from requests to req_completed
        mv(dir, filename);
        console.log("~~~ Request was successful! request designated as completed.");
    } else {
        console.error("~~~ Request was unsuccessful, re-trying...");
    };

    res.send('hi GET');
});


// UI request is handled here (creation of the requests):
app.post('/data', function (req, res) {
        // Extract the required values from the JSON file:
        var data = req.body;

        var ext = data["name"]["crowdshot_id"];

        // Write the data in the request to a JSON file
        var file = './requests/' + ext;
        jsonfile.writeFile(file, data, function(err){
                // Update the JSON file and send back the request
                if (err != null) {
                        data["name"]["status"] = "queuefailed";
                        console.error(err);
                } else {
                        data["name"]["status"] = "queued";
                        console.log('json file ' + file + ' created and queued');
                }
                res.send(data);
        });

        /*
    //Custom Header pass
    var headersOpt = {
        "content-type": "application/json",
    };
        request(
         {
             method:'POST',
             url:'http://xxx.xx.xxx.xx.xip.io:8080/data',
             body: data,
             headers: headersOpt,
             json:true,
         },  function (error, response, body) {
             //Print the Response
             console.log(data);
             //console.log(response);
             console.log(body);
             //console.log(error);
             //res.send(body);
         });

        */

    //res.send('Application control passed to the processing server...');

});

// set a timer and send requests to the tech server



// Initate the watcher to look over the requests
var watcher = chokidar.watch('./requests', {ignored: /[\/\\]\./, persistent: true})
var path = './requests';
watcher
  //.on('add', function(path) {console.log('File', path, 'has been added');})
  .on('addDir', function(path) {console.log('Directory', path, 'has been added');})
  .on('change', function(path) {console.log('File', path, 'has been changed');})
  .on('unlink', function(path) {console.log('File', path, 'has been removed');})
  .on('unlinkDir', function(path) {console.log('Directory', path, 'has been removed');})
  .on('error', function(error) {console.error('Error happened', error);})


watcher.on('change', function(path, stats) {
  console.log('Directory', path, 'changed size to', stats.size);
});


const server = http.createServer(app).listen(8080, function(err) {
  if (err) {
    console.log(err);
  } else {
    const host = server.address().address;
    const port = server.address().port;
    console.log(`Server listening on ${host}:${port}`);
  }

});


