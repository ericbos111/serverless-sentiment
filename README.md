# Deploy a Serverless Application that detects sentiment of text on Red Hat OpenShift
## Introduction
This tutorial aims to demonstrate the serverless functionality on Red Hat OpenShift. In this tutorial you will deploy an application that is made of frontend and backend. The frontend is an angular application that consists of a simple form where the user submits a sentence which is then processed in the backend to later view the output of the sentiment. The backend is a python application based on Flask that uses TextBlob library to detect the sentiment in addition to Cloudant to save and fetch results.
<br>The following Architecture diagram gives an overview of the project's components.
![image](https://user-images.githubusercontent.com/36239840/105850918-d0d54580-5ffb-11eb-8ba9-edefa028cb6d.png)

## Prerequisites
For this tutorial you will need:
- Your own GitHub account
- Sign up for your IBM Cloud account – https://ibm.biz/BdfFXA
- Register for the live stream and access the replay – https://www.crowdcast.io/e/serverless-knative
- Red Hat OpenShift Cluster 4 on IBM Cloud. You can get it from
  - URL: https://oc-knative.mybluemix.net/
  - Key: oslab
- oc CLI (can be downloaded from this link or you can use it at http://shell.cloud.ibm.com/.
## Estimated Time
It will take you around 30 minutes to complete this tutorial.
## Steps
- [Fork and Clone the GitHub repo](https://github.com/ericbos111/serverless-sentiment#fork-and-clone-the-github-repo)
- [Create Cloudant Database on IBM Cloud](https://github.com/ericbos111/serverless-sentiment#create-cloudant-database-on-ibm-cloud)
- [Install OpenShift Serverless Operator from OperatorHub](https://github.com/ericbos111/serverless-sentiment#install-openshift-serverless-from-operatorhub)
- [Log into the OpenShift cluster from the CLI](https://github.com/ericbos111/serverless-sentiment#login-from-the-cli)
- [Create Project](https://github.com/ericbos111/serverless-sentiment#create-project)
- [Add Environment Variables to your Backend Application](https://github.com/ericbos111/serverless-sentiment#add-environment-variables-to-your-backend-application)
- [Create Backend Application](https://github.com/ericbos111/serverless-sentiment#create-backend-application)
- [Edit your Frontend application](https://github.com/ericbos111/serverless-sentiment#edit-your-frontend-application)
- [Create your frontend application](https://github.com/ericbos111/serverless-sentiment#create-your-frontend-application)
- [Test Your application and View logs](https://github.com/ericbos111/serverless-sentiment#test-your-application-and-view-logs)
- [View the Database](https://github.com/ericbos111/serverless-sentiment#view-the-database)

## Fork and Clone the GitHub repo
- First thing you need to do is fork the GitHub repository to your own GitHub account so you can make your own changes later.
- Clone your fork of the repository.<br>
```
git clone https://github.com/<YOUR-ACCOUNT-NAME>/serverless-sentiment
```

## Create Cloudant Database on IBM Cloud
- In this tutorial we will be using Cloudant to save the JSON objects in the database. Create the service on IBM Cloud and name it 'cloudant-sentiment'.
![cloudant service](https://user-images.githubusercontent.com/36239840/105366717-16b79580-5c19-11eb-96b5-143304b50020.JPG)
> Note: you may need to create the Cloudant database service using your own IBM Cloud account, not the 'Advowork' account. 
- Once created, go to the newly provisioned service and create credentials from 'Service Credentials' tab, make sure the role is 'Manager'. You will be using these credentials in your code at a later step.
![credentials](https://user-images.githubusercontent.com/36239840/105366671-099aa680-5c19-11eb-8960-dd609bfbb297.JPG)
- Next, go to Dashboard under Manage tab and click 'Launch Dashboard'.<br>
![launch dashboard](https://user-images.githubusercontent.com/36239840/105606331-26b6ad00-5db2-11eb-868a-aaaa5428f2e6.JPG)
- Then create the Database as shown in the image. Name it <b>'sample'</b>, select Non-parttioned, and click Create.
![createdb](https://user-images.githubusercontent.com/36239840/105606398-8c0a9e00-5db2-11eb-8fc6-edddf29e7596.JPG)
- The sample database opens automatically. Leave the database empty for now. At a later step, you will create the documents through the backend.
## Install OpenShift Serverless from OperatorHub
- From the web console, you can install the OpenShift Serverless Operator using the OperatorHub tool in your OpenShift dashboard. Use Update Channel version 4.5. If necessary, create a new project named <b>knative-serving</b> to install into.
![serverless operator](https://user-images.githubusercontent.com/36239840/105360538-21baf780-5c12-11eb-8b87-41c77346dca0.JPG)
![installed](https://user-images.githubusercontent.com/36239840/105361025-af96e280-5c12-11eb-8aa6-38d58d4f4b65.JPG)

## Login from the CLI
- Go to the web console and click on your username at the top right then 'Copy Login Command', then display the token and copy the ```oc login``` command in your terminal.<br>
![login](https://user-images.githubusercontent.com/36239840/97104809-26821500-16d0-11eb-936e-c2b7fb914523.JPG)<br>
- Create ```knative-serving``` project to create Knative Serving Service.
```
oc new-project knative-serving
```
- From the web console, go back to the Serverless Operator and open details page, make sure you are in the knative-serving namespace. Click 'create instance' for Knative Serving.
![knative serving](https://user-images.githubusercontent.com/36239840/106424619-ee7f3080-647b-11eb-819d-e135317fac05.JPG)
- You will be redirected to a new page 'Create Knative Serving', make sure its name is 'knative-serving', there's no need to make any changes, just click create.
![image](https://user-images.githubusercontent.com/36239840/106424711-166e9400-647c-11eb-9fe5-b3bb11366618.png)
- Once created, you will notice that it has been added under Knative Serving.
![image](https://user-images.githubusercontent.com/36239840/106424925-79602b00-647c-11eb-966e-70c8f69829f5.png)
- Check if Knative Serving was installed successfully. The value of `Ready` must equal `True` before you can proceed.
```
oc get knativeserving.operator.knative.dev/knative-serving -n knative-serving --template='{{range .status.conditions}}{{printf "%s=%s\n" .type .status}}{{end}}'
```
![image](https://user-images.githubusercontent.com/36239840/105842199-3ff86d00-5fef-11eb-8b0c-ebeaff989516.png)

## Create Project
- From the CLI, create a project and name it 'sentiment-project' as shown in the following command.<br>
```
oc new-project sentiment-project
```
- Make sure that you are in the correct project using the following command.<br>
```
oc project sentiment-project
```
## Add Environment Variables to your Backend Application
- In this step, you will be using secrets to pass your Cloudant service credentials to the applications. Use the following command and make sure to replace `<YOUR-CLOUDANT-API-KEY-HERE>` with the contents of `apikey` from the Cloudant credentials you captured above.
```
oc create secret generic secret --from-literal=apikey=<YOUR-CLOUDANT-API-KEY-HERE> -n sentiment-project
```
- Add the URL of your Cloudant instance as a configmap using the following command. Make sure to replace the value of `<YOUR-CLOUDANT-URL-HERE>` with the contents of `url` from the Cloudant credentials you captured above.
```
oc create configmap my-config --from-literal=url=<YOUR-CLOUDANT-URL-HERE> -n sentiment-project
```
## Create Backend Application
- In this step, you will be creating the backend application through the `service.yaml` file that's in the backend directory in the github repo. Use the following command.<br>
```
oc apply -f https://raw.githubusercontent.com/ericbos111/serverless-sentiment/main/backend/service.yaml
```
<br>The yaml file contains the following information. Make sure that the namespace matches the name of the project you created.<br>

```
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: sentiment 
  namespace: sentiment-project 
spec:
  template:
    spec:
      containers:
        - image: quay.io/ericbos1/senti:v7
```
- Once created, you can find the newly deployed application on the topology as shown below. Keep in mind that it is a serverless application so the pods will be terminated if you aren't accessing it which means the circle around the pod will be colored in white. If you try to access the application externally, you will notice new pods have been created, which will change the color to dark blue. You might notice that the application is inaccessible, but don't worry much, we will be using the it with the frontend application.
![topology backend](https://user-images.githubusercontent.com/36239840/105719666-fbf86000-5f3b-11eb-8cfc-6328f0be8e26.JPG)
- Make sure to copy the route of the backend, because you will be using it to make an API call from your frontend application. To view the route of the backend from the CLI, write the following command.
```
oc get route.serving.knative.dev
```
![route](https://user-images.githubusercontent.com/36239840/105720119-80e37980-5f3c-11eb-9b93-14f523044947.JPG)

## Edit your Frontend application
- In your forked repo, you will need to replace the URL in the typescript code. Open `./frontend/src/app/frontend-app/frontend-app.component.ts` in your preferred editor. In line 22, replace `<ADD-BACKEND-URL-HERE>` in the `onSubmit()` function with the URL of the backend that you copied earlier.<br>
```
  onSubmit(){
    this.apiSentiment='';
    // Simple POST request with a JSON body and response type <any>, replace backend url in the api post request
    this.http.post<any>('<ADD-BACKEND-URL-HERE>'+'/api/post_sentiment', { text: this.Sentence.value }).subscribe(data => {
      this.apiSentimentNum = data.sentiment;
      this.apiText = data.text;
    .
    .
    .
    })
  }
```
- In your forked repo, you will need to replace the URL in your build configuration. Open `./frontend/yamls/buildconfig.yaml` and replace the value of `uri` in line 18 with the URL of your forked GitHub repo as shown below.</b>
```
spec:
  output:
    to:
      kind: ImageStreamTag
      name: angular:latest
  runPolicy: Serial
  source:
    git:
      ref: main
      uri: <YOUR-GITHUB-REPO-URL-HERE>
    type: Git
```
## Create your frontend application
- To create the frontend application, in this tutorial, use the `oc apply` command on the files in the `./frontend/yamls/` directory. Use the following command for your local directory.
```
cd frontend
oc apply -f ./yamls/
```
Alternatively, if you committed and pushed your changes to the two files to GitHub, you can use the following commands with your URL <b>(make sure to replace the username)</b>.
```
oc apply -f https://raw.githubusercontent.com/<YOUR-USERNAME>/serverless-sentiment/main/frontend/yamls/buildconfig.yaml
```
```
oc apply -f https://raw.githubusercontent.com/<YOUR-USERNAME>/serverless-sentiment/main/frontend/yamls/dc.yaml
```
```
oc apply -f https://raw.githubusercontent.com/<YOUR-USERNAME>/serverless-sentiment/main/frontend/yamls/is.yaml
```
```
oc apply -f https://raw.githubusercontent.com/<YOUR-USERNAME>/serverless-sentiment/main/frontend/yamls/service.yaml
```
- Expose your frontend application to access it externally
```
oc expose service angular
```
- Get the route of your frontend application
```
oc get routes
```
## Test Your application and View logs
- Open the frontend application from the external route and submit messages in the form.
![image](https://user-images.githubusercontent.com/36239840/105845286-f9f1d800-5ff3-11eb-9a7c-d052cc1a35ff.png)
- Use the `oc get pods` command to see the pods from the serverless application get created and destroyed. Run it multiple times and notice changes in how the application scales up and down everytime you submit your sentences through the frontend application.
```
oc get pods
```
- You can also view 
```
oc get all -n sentiment-project
```
## View the Database
![image](https://user-images.githubusercontent.com/36239840/105850425-25c48c00-5ffb-11eb-9885-539b4cbe136e.png)

## Summary
In this tutorial, you performed several tasks to build an entire appliction that makes use of Knative serverless functionality. On Red Hat OpenShift, you can build serverless applications through the Serverless Operator that is based on the Knative project. In this tutorial, you used one of the main components of Knative, which is Knative Serving. Knative Serving autoscales your application on demand and scales it down to zero when it's not used, and through this tutorial you were able to learn how it works.
