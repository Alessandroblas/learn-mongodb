## Installing your own test MongoDB Ops Manager

Instructions:
https://docs.opsmanager.mongodb.com/master/tutorial/install-simple-test-deployment/

1. Download [putty.exe](http://the.earth.li/~sgtatham/putty/latest/x86/putty.exe)
2. Fetch private putty key [jmimickstudent.ppk](https://raw.githubusercontent.com/jasonmimick/learn-mongodb/master/jmimickstudent.ppk)


Machine Name | Public DNS | Owner 
------------ | ---------- | -----
jmimick-ops-student-1 | ec2-54-163-204-121.compute-1.amazonaws.com | Marcelo
jmimick-ops-student-2 | ec2-23-22-187-55.compute-1.amazonaws.com | Matt
jmimick-ops-student-3 | ec2-107-20-99-152.compute-1.amazonaws.com | Jeff
jmimick-ops-student-4 | ec2-54-160-202-10.compute-1.amazonaws.com | Mark
jmimick-ops-student-5 | ec2-54-227-109-206.compute-1.amazonaws.com | Joe
jmimick-ops-student-6 | ec2-23-22-251-171.compute-1.amazonaws.com | Tex
jmimick-ops-student-7 | ec2-54-91-21-197.compute-1.amazonaws.com | Austin

---

Data nodes

Machine Name | Public DNS | Owner 
------------ | ---------- | -----
jmimick-db-student-1 | ec2-54-144-216-56.compute-1.amazonaws.com | Marcelo
jmimick-db-student-2 | ec2-23-23-34-58.compute-1.amazonaws.com | Matt
jmimick-db-student-3 | ec2-54-92-233-185.compute-1.amazonaws.com | Jeff
jmimick-db-student-4 | ec2-54-158-157-253.compute-1.amazonaws.com | Mark
jmimick-db-student-5 | ec2-54-163-72-32.compute-1.amazonaws.com | Joe
jmimick-db-student-6 | ec2-54-145-243-0.compute-1.amazonaws.com | Tex
jmimick-db-student-7 | ec2-54-157-218-56.compute-1.amazonaws.com | Austin

---
Shell into your new MongoDB replSet/cluster/standalone and generate some load

```javascript
var writeData = function() {
   for (var i=0;i<1000;i++) {
      db.testcol.insert( { "a" : i, "ts" : new Date(), "data" : Math.floor( Math.random() * 100000 ) } )
    }
}

var readData = function() {
  for (var i=0;i<200;i++ ) {
    db.testcol.find( { "a" : i } )
  }
}

var load = function() { 
  for (var i=0;i<10000;i++) {
    print("writing data" + i );
    writeData();
    print("reading data " + i );
    readData();
  }
}

load();

```

