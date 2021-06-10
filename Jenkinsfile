
import groovy.json.JsonSlurper
import groovy.json.JsonOutput; 

def CONFIGDETAILS 
String MISSINGQC = ""
def INSTANCEARN = ""
def TRAGETINSTANCEARN = ""
String PRIMARYQC = ""
String TARGETQC = ""
String PRIMARYQUEUES = ""
String TARGETQUEUES = ""
String PRIMARYCFS = ""
String TARGETCFS = ""
String PRIMARYHOP = ""
String TARGETHOP = ""

pipeline {
    agent any
    stages {
        stage('git repo & clean') {
            steps {
                script{
                   try{
                      sh(script: "rm -r ac-queue-syncronization", returnStdout: true)    
                   }catch (Exception e) {
                       echo 'Exception occurred: ' + e.toString()
                   }                   
                   sh(script: "git clone https://github.com/ramprasadsv/ac-queue-syncronization.git", returnStdout: true)
                   sh(script: "ls -ltr", returnStatus: true)
                   CONFIGDETAILS = sh(script: 'cat parameters.json', returnStdout: true).trim()
                   def config = jsonParse(CONFIGDETAILS)
                   INSTANCEARN = config.primaryInstance
                   TRAGETINSTANCEARN = config.targetInstance
                }
            }
        }
        
        stage('List all Resources') {
            steps {
                echo "List all Resources in both instance "
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {
                    script {
                        
                        PRIMARYQUEUES =  sh(script: "aws connect list-queues --instance-id ${INSTANCEARN} --queue-types STANDARD", returnStdout: true).trim()
                        echo PRIMARYQUEUES
                        TARGETQUEUES =  sh(script: "aws connect list-queues --instance-id ${TRAGETINSTANCEARN} --queue-types STANDARD", returnStdout: true).trim()
                        echo TARGETQUEUES
                        
                        PRIMARYQC =  sh(script: "aws connect list-quick-connects --instance-id ${INSTANCEARN}", returnStdout: true).trim()
                        echo PRIMARYQC
                        TARGETQC =  sh(script: "aws connect list-quick-connects --instance-id ${TRAGETINSTANCEARN}", returnStdout: true).trim()
                        echo TARGETQC 
                        
                        PRIMARYCFS =  sh(script: "aws connect list-contact-flows --instance-id ${INSTANCEARN} --contact-flow-types OUTBOUND_WHISPER", returnStdout: true).trim()
                        echo PRIMARYCFS
                        TARGETCFS =  sh(script: "aws connect list-contact-flows --instance-id ${TRAGETINSTANCEARN} --contact-flow-types OUTBOUND_WHISPER", returnStdout: true).trim()
                        echo TARGETCFS
                      
                        PRIMARYHOP = sh(script: "aws connect list-hours-of-operations --instance-id ${INSTANCEARN}", returnStdout: true).trim()
                        echo PRIMARYHOP
                        TARGETHOP = sh(script: "aws connect list-hours-of-operations --instance-id ${TRAGETINSTANCEARN}", returnStdout: true).trim()
                        echo TARGETHOP                      
                    }
                }
            }
        }
        
        stage('Find missing queues') {
            steps {
                script {
                    echo "Find missing queues in the target instance"
                    def pl = jsonParse(PRIMARYQUEUES)
                    def tl = jsonParse(TARGETQUEUES)
                    int listSize = pl.QueueSummaryList.size() 
                    println "Primary list size $listSize"
                    for(int i = 0; i < listSize; i++){
                        def obj = pl.QueueSummaryList[i]
                        String qcName = obj.Name
                        String qcId = obj.Id
                        boolean qcFound = checkList(qcName, tl)
                        if(qcFound == false) {
                            println "Missing Name : $qcName Id : $qcId"                                                              
                            MISSINGQC = MISSINGQC.concat(qcId).concat(",")                                
                        }
                    }
                }
                echo "Missing list in the target instance -> ${MISSINGQC}"
            }
        }
        
        stage('Create the missing queues') {
            steps {
                echo "Create the missing queues in the target instance "                
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {   
                    script {
                        if(MISSINGQC.length() > 1 ){
                            def qcList = MISSINGQC.split(",")
                            for(int i = 0; i < qcList.size(); i++){
                                String qcId = qcList[i]
                                if(qcId.length() > 2){
                                    def di =  sh(script: "aws connect describe-queue --instance-id ${INSTANCEARN} --queue-id ${qcId}", returnStdout: true).trim()
                                    echo di
                                    def dq =  sh(script: "aws connect list-queue-quick-connects --instance-id ${INSTANCEARN} --queue-id ${qcId}", returnStdout: true).trim()
                                    echo dq
                                    def qc = jsonParse(di)
                                    def quickConnectList = jsonParse(dq)
                                    def quickConnectList  
                                    String targetQCList
                                    if(quickConnectList) {
                                        for(int j=0; j< quickConnectList.QuickConnectSummaryList.size(); j++) {
                                            def obj = quickConnectList.QuickConnectSummaryList[j]
                                            String newId = getQuickConnectId(PRIMARYQC, obj.Id, TARGETQC)
                                            targetQCList = targetQCList.concat(newId)
                                        }
                                    }
                                    echo "QC List -> ${targetQCList}"

                                    String qcName = qc.Queue.Name
                                    String qcDesc = qc.Queue.Description
                                    String ouboundFlowId 
                                    String qcCallerName
                                    if(qc.Queue.OutboundCallerConfig) {
                                        if(qc.Queue.OutboundCallerConfig.OutboundFlowId) {
                                            ouboundFlowId = getFlowId (PRIMARYCFS, qc.Queue.OutboundCallerConfig.OutboundFlowId, TARGETCFS)  
                                        }
                                        if(qc.Queue.OutboundCallerConfig.OutboundCallerIdName ) {
                                            qcCallerName = qc.Queue.OutboundCallerConfig.OutboundCallerIdName 
                                        }                                                                      
                                    }

                                    String hopId
                                    if(qc.Queue.HoursOfOperationId) {
                                        hopId = getHopId (PRIMARYHOP, qc.Queue.HoursOfOperationId, TARGETHOP)
                                    }
                                    String maxContacts
                                    if(qc.Queue.MaxContacts ) {
                                        maxContacts = qc.Queue.MaxContacts 
                                    }
                                    String status = qc.Queue.Status 
                                    qc = null

                                    //String qcConfig = "QuickConnectType=QUEUE,QueueConfig=\\{QueueId=" + targetQueueId + ",ContactFlowId=" + targetFlowId +"\\}"                                    
                                    //def cq =  sh(script: "aws connect create-quick-connect --instance-id ${TRAGETINSTANCEARN} --name ${qcName} --description ${qcDesc} --quick-connect-config ${qcConfig}" , returnStdout: true).trim()
                                    //echo cq

                               }
                            }
                        }
                    }                
                }
            }
        } 

         stage('Create Missing quick connects') {
            steps {
                echo "Create the quick connects that were missing"                
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {   
                }
            } 
         }
        
     }
}


@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurper().parseText(json)
}

def toJSON(def json) {
    new groovy.json.JsonOutput().toJson(json)
}

def checkList(qcName, tl) {
    boolean qcFound = false
    for(int i = 0; i < tl.QueueSummaryList.size(); i++){
        def obj2 = tl.QueueSummaryList[i]
        String qcName2 = obj2.Name
        if(qcName2.equals(qcName)) {
            qcFound = true
            break
        }
    }
    return qcFound
}

def getFlowId (primary, flowId, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    println "Searching for flowId : $flowId"
    for(int i = 0; i < pl.ContactFlowSummaryList.size(); i++){
        def obj = pl.ContactFlowSummaryList[i]    
        if (obj.Id.equals(flowId)) {
            fName = obj.Name
            println "Found flow name : $fName"
            break
        }
    }
    println "Searching for flow name : $fName"        
    for(int i = 0; i < tl.ContactFlowSummaryList.size(); i++){
        def obj = tl.ContactFlowSummaryList[i]    
        if (obj.Name.equals(fName)) {
            rId = obj.Id
            println "Found flow id : $rId"
            break
        }
    }
    return rId
}

def getQuickConnectId (primary, qcId, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    println "Searching for qcId : $qcId"
    for(int i = 0; i < pl.QuickConnectSummaryList.size(); i++){
        def obj = pl.QuickConnectSummaryList[i]    
        if (obj.Id.equals(qcId)) {
            fName = obj.Name
            println "Found qc name : $fName"
            break
        }
    }
            
    for(int i = 0; i < tl.QuickConnectSummaryList.size(); i++){
        def obj = tl.QuickConnectSummaryList[i]    
        if (obj.Name.equals(fName)) {
            rId = obj.Id
            println "Found flow id : $rId"
            break
        }
    }
    return rId
    
}

def getHopId (primary, userId, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    println "Searching for userId : $userId"
    for(int i = 0; i < pl.UserSummaryList.size(); i++){
        def obj = pl.UserSummaryList[i]    
        if (obj.Id.equals(userId)) {
            fName = obj.Username
            println "Found user name : $fName"
            break
        }
    }
    println "Searching for userId for : $fName"        
    for(int i = 0; i < tl.UserSummaryList.size(); i++){
        def obj = tl.UserSummaryList[i]    
        if (obj.Username.equals(fName)) {
            rId = obj.Id
            println "Found flow id : $rId"
            break
        }
    }
    return rId
    
}

