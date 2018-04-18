# Roughbook
Why we created DSL when jenkins file exist
There are some options in ahop-job in inova instance like recursive or snapshot dependencies. There are not pre defined api in Jenkins file
So DSL is another approach with which we can create a script for our Jenkins job.
# Issues observed:
Once we increased our Ram size from 512MB to 1GB, sample job was able to progress but failed at nabu step that is failing to find a file that should be part of Repository
Cloudfees nabu folder is not getting cloned so we manually cloned the reposiotry to get the build succeed.
# Created a Docker file
Create a Docker file that will create a custom image with Checkmarx plugin installed in it. Also make modfiyications to create an image which contains jobs in it
For sample, created some config test files in config folder
Create the image with custom defined plugins and Jobs
use docker build -t "custom-jenkins" .
that creats a docker image
Make this image available in our Open shift image registry
Creat a deployment yaml that makes use of this image .
Automate the container deployment through a shell script
Jenkins persistent images is predifined with volume 1 Gb for /var/lib/jenkins i.e, Jenkins home.
There is no option to modify the volume size from webconsole
So I see two options:
See if we can update the volume from command line using oc command line tools . We need to redploy jenkins after update to the volume configuration
Or create a new mount point with defined sizeand with the help of environment variables using config maps, use the new mount point as Jenkins home.
# Commands to list volumes: (Donot perform oc rsh )
Logon on to project oc project <project-name>
oc get pv (lists of all available persistent volumes configured)
To claim a suitable persistent volume and mount it using oc volume dc/nginx --add --claim-size 512M --mount-path /usr/share/nginx/html --name downloads
oc get pv will list the new persistent volume claim attached to one of the peristent volumes available
oc get pvc to list all persistent volume claims
oc volume dc --all to which applications the pvc are attached to.
oc volume dc/nginx --remove --name downloads to deattach the volumes . This will not remove the pvc from the list. The command will deattach/unmount the volume from nginx application. So that the conatiner cannot use the mount point
oc volume dc --all
oc volume dc/nginx --add --type pvc --claim-name pvc-rkly7 --mount-path /usr/share/nginx/html --name downloads remount the same pvc into the applicaiton.
oc volume dc/nginx --remove --name downloads and oc delete pvc/pvc-rkly7 to detach and delete the pvc completely from pvc list. If so the complete data will be lost.
oc describe pod <podname> to get the pod details on the console
oc get pod <podname> -o yaml to get the details in yaml output
oc rsh <podname>
mount
df -h
cd /var/lib/jenkins
du -h --max-depth=2
# Note: dont run steps 3,7,9,10 . And for the remaining commands please send the output
oc get pvc to list all persistent volume claims in the name space. This will o/p the pvc name, size and mount path.
Only jenkins pvc is avilable in the name space and is having 1Gi capacity.
First option: Tried resizing the pvc using oc volume dc/jenkins --add --overwrite --type pvc --claim-name jenkins -- claim-size 2Gi . But oc is not allwoing it to resize the volume with forbidden error.
Second option: oc get pvc jenkins -o yaml > pvc.yaml . Get the pvc configuration in yaml format and redirect it to pvc.yaml.
Modify the capacity key from 1Gi to 2Gi in pvc.ymal file
oc create -f pvc.yaml. using oc create tried to create a pvc object, but this also failed with similar error.
Third option: oc edit pvc jenkins. Tried to modify the capacity from pvc object, but even this failed with error. The error says once the pvc is created it is immutable. SO we cannot resize the pvc.
So as all the options are tried out, the only option I see right now is to recreate a new deployment configuration with defined size.
Also check with the team if there is any option to resize the existing jenkins pvc.
Create a new mount to store the .m2 configuration. As of now by default the .m2 is getting stored at /var/lib/jenkins.
The steps would be to create a new mount named maven and mount it to /maven/ i. oc get pvc ii. Create a new pvc object for maven
 
iii. Using this claim as volume in the pods yaml apiVersion: "v1" kind: "Pod" metadata: name: "mypod" labels: name: "frontendhttp" spec: containers: - name: "myfrontend" image: "nginx" ports: - containerPort: 80 name: "http-server" volumeMounts: - mountPath: "/var/www/html" name: "maven" volumes: - name: "maven" persistentVolumeClaim: claimName: "maven" 3. Modify the maven configuration to store the repositories in the new mount point. 4. Usually maven installed through Jenkins will be placed at <JENKINS_HOME>/tools.
references: https://docs.openshift.com/enterprise/3.1/dev_guide/persistent_volumes.html
# oc rsync:
To copy a single file from the container to the local machine, the form of the command you need to run is:
oc rsync <pod-name>:/remote/dir/filename ./local/dir
Uploading files from a conatiner:
oc rsync ./local/dir pod-name:/remote/dir
we can use --exclude=* and --include=filename and also --delete to have the both folders in sync and also --no-perms to forget about permissions ref: https://blog.openshift.com/transferring-files-in-and-out-of-containers-in-openshift-part-1-manually-copying-files/
# Jenkins Credentails Automation:
There are two options to automate the credentials addition in Jenkins.
First option: To store password in plain text in credentials.xml, copy it over to jenkins machine after installing and starting the service. Then jenkins will encrypt it with it's new secret and it will work Second option: A second option is to install jenkins, start it and then copy the credentials.xml with encrypted passwords together with secrets directory and secret.xml from previous instllation. This will copy both encryption master key and the encrypted credentials that have been created using this master key.
Copy the credentials.xml file, secrets directory and secrets.xml from working pod to local using rsync.
# How to copy the above files to pod as part of automatic deployment.
The files should be mounted once the /var/lib/jenkins mount point is available.
Store the files in a private git repository in github internal
Using pod life cycle hooks, clone the repo on to the pod and also execute a command to copy that to /var/lib/jenkins.Option 1: Directly update the dc using oc edit dc jenkins and update the spec with post
 
Deploy the changes using oc rollout latest jenkins
Option 2: Using oc patch from client. create run.sh script and store it in a repo. run.sh contains the below code.
 
Source run.sh to run the script to perform a patch operation oc rollout latest jenkins to deploy the latest dc
or Use rsync to copy the files as it is time process
Can we make use of config maps? Investigate further..
ref: https://blog.openshift.com/using-post-hook-to-initialize-a-database/
Failures:
Jenkins persistent is having issues and is going down frequently . So opted for ephermal jenkins image with a volume mount of 100Gi
Created a config map for saxon lic and added that as part of the deployment config yaml file,upon rollout deployment was getting failed. Tried troublesshooting but couldn't trace out the issue. The same worked with Persistent image
Java/maven is not configured and also no DSL plugin is installed.
The idea to install plugins is to create a DSL plugin binary DSL.hpi file from source code and then use jenkins cli upload it to plugins.
When tried to install maven using configrue jenkins, maven is not getting installed. So had to manaullay install the maven using oc rsync to copy the binary from local machine to pod and extract it, set maven home and path settings
Even java is not configured properly. Need to setup this
Notes:
Most of the times, not able to launch the jenkins url and after sometime it automatically gets up . We don't see any pod/service/route status as down . Find out how to fix this issue
and also need to create a new mount point to store .m2 folder.
dont have access to get the volumes oc get pv
Which volume should be used to create a new mount. How to get the list of volumes and which one we should opt for.
