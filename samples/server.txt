import flask
from flask import request
from flask import url_for
from flask import jsonify # For AJAX transactions

import json
import logging

import random
import math
# Mongo database
import pymongo
from pymongo import MongoClient
from bson.objectid import ObjectId

# Date handling
import arrow # Replacement for datetime, based on moment.js
#import datetime # But we still need time
#from dateutil import tz  # For interpreting local times

###
# Globals
###
app = flask.Flask(__name__)
import CONFIG


import uuid
app.secret_key = str(uuid.uuid4())
app.debug = CONFIG.DEBUG
app.logger.setLevel(logging.DEBUG)

with open('static/questions.txt','r') as inf:
    questions = eval(inf.read())
try:
    dbclient = MongoClient(CONFIG.MONGO_URL)
    db = dbclient.service
    collectionClassDB = db.classDB
    collectionAccounts = db.adminAccounts
    collectionFormsDB = db.forms
except:
    print("Failure opening database.  Is Mongo running? Correct password?")
    #sys.exit(1)

#############
####Pages####
#############


### Home Page ###

@app.route("/")
@app.route("/index")
def index():
    app.logger.debug("Main page entry")
    return flask.render_template('index.html')

### Client Page ###

@app.route("/client")
def client():
    app.logger.debug("client page entry")
    if flask.session.get("questions") == None:
        flask.session['questions'] = questions
    if flask.session.get('priorityList') == None:
        flask.session['priorityList'] = None
### ADDED OPTION FOR TWO PAGES FOR FORMS ###
### ONE PRE PRIORITY, ONE POST ###

    if flask.session.get('priorityList') == None:
        return flask.render_template('client.html')
    else:
        return flask.render_template('client.html')

### Admin Page ###

@app.route("/admin")
def admin():
    app.logger.debug("admin page entry")
    app.logger.debug(flask.session.get('login'))
    if flask.session.get('login') == None:
        return flask.render_template('login.html')
    classList = collectionAccounts.find_one({"_id":ObjectId(flask.session.get('login'))}).get("classList")
    accList = []
    for aId in classList:
        accList.append(str(collectionClassDB.find_one({"_id":ObjectId(aId)}).get("_id")))
    flask.session['myClassDB'] = accList
    return flask.render_template('admin.html')

@app.route("/adminCreate")
def adminCreate():
    app.logger.debug("admin create page entry")
    return flask.render_template('adminCreate.html')

@app.route('/login')
def login():
    if flask.session.get('login') != None:
        return flask.redirect(url_for('admin'))
    else:
        return flask.render_template('login.html')

@app.route('/createAdmin')
def createAdmin():
        return flask.render_template('createAdmin.html')

@app.errorhandler(404)
def page_not_found(error):
    app.logger.debug("Page not found")
    flask.session['linkback'] = flask.url_for("index")
    return flask.render_template('page_not_found.html'), 404

################
###functions###
################

###for index page###
@app.route("/_createTeams")
def preTeamCreate():
    classId = request.args.get('classID',0, type=str)
    groupSizeMax = request.args.get('groupSizeMax',0, type=int)
    d = createTeams(classId,groupSizeMax);
    print(d)
    #d = "yes"
    return jsonify(result = d)


@app.route("/_portal")
def portalSelector():
    objId = request.args.get('portal', 0, type=str)
    app.logger.debug(objId)
    if objId == "client":
        return flask.redirect(url_for('client'))
    else:
        return flask.redirect(url_for('admin'))

###for login page###
@app.route("/_submitLoginRequest")
def loginGate():
    adminName = request.args.get('adminName',0,type=str)
    adminKey = request.args.get('adminKey',0,type=str)
    account = collectionAccounts.find_one({'name':adminName,'password':adminKey})
    app.logger.debug(account)
    if account != None:
        flask.session['login'] = str(account.get('_id'))
        flask.session['name'] = account.get('name')
        d = {'result':'success'}
    else:
        d = {'result':'failed'}
    d = json.dumps(d)
    return jsonify(result = d)

### for administrative settings###

##
#    requests:
#       setting
#           "addAdmin"
#               get "adminName"
#               return "added"||"account exists"
#           "removeAdmin"
#               get "AdminId"
#               return redirect to login
#          "logout"
#               return redirect to login
#


@app.route("/_adminSettings")
def adminSettings():
    setting = request.args.get('setting',0,type=str)
    adminName = request.args.get('adminName',0,type=str)
    adminKey = request.args.get('adminKey',0,type=str)
    print adminName
    if setting  == "addAdmin":
        classList = [] #a list of team builder object ids for the administrator
        if collectionAccounts.find_one({'name':adminName}) == None:
            record = {"name":adminName, "password":adminKey, "date":arrow.utcnow().naive, "classList": classList}
            collectionAccounts.insert(record)
            d = "added"
            print d
        else:
            d = "account exists"
    elif setting == "removeAdmin":
        aClassList = collectionAccounts.find_one({"_id":ObjectId(flask.session.get('login'))}).get("classList")
        for aVal in aClassList:
            deleteForms(aVal)
        collectionAccounts.remove({'_id':ObjectId(flask.session.get('login'))})
        removeState()
        return flask.redirect(url_for('login'))
    elif setting == "logout":
        removeState()
        return flask.redirect(url_for('login'))
    else:
        d = "wat"
    print d
    return jsonify(result = d)

def removeState():
    flask.session['name'] = None
    flask.session['login'] = None
    flask.session['myClassDB'] = None


###############################################################################


##
#    requests:
#       setting
#           "addClass"
#               get "className"
#               return "added"
#           "removeClass"
#               get "classId"
#               return "removed" | "failed" (for future)
#          "setPriorities"
#              get "classId"
#              return "priorty updated"
#

@app.route("/_classDBSettings")
def preSettings():
    aReturn = request.args.get('aThing',0,type=str)
    aVal = json.loads(aReturn)
    setting = aVal.get("setting")
    className = aVal.get("className")
    classId = aVal.get("classID")
    priorityList = aVal.get("priorityList")
    if setting != "removeClass":
        for x in range(0,len(priorityList)):
            if priorityList[x] == None:
                priorityList[x] = 0
            else:
                priorityList[x] = int(priorityList[x])
    d = classDBSettings(setting,className,classId,priorityList)
    return jsonify(result = d)

def classDBSettings(setting,className,classId,priorityList):
    if setting == "addClass":
        #qPriority = [0,0,0,0,0,0,0,0,0,0]
        aTime = arrow.utcnow().naive
        formList = []
        record = {"name": className, "date":aTime , "formList":formList, "qPriority":priorityList}
        collectionClassDB.insert(record)
        aClass = collectionClassDB.find_one({"date": aTime})
        locId = str(aClass.get('_id'))
        locList = collectionAccounts.find_one({"_id":ObjectId(flask.session.get('login'))}).get("classList")
        locList.append(locId)
        collectionAccounts.update_one(
            {"_id": ObjectId(flask.session.get('login'))},
            {"$set": {"classList":locList}}
        )
        d = "added"
    elif setting == "removeClass":
        locList = collectionAccounts.find_one({"_id":ObjectId(flask.session.get('login'))}).get("classList")
        ### add checker for if ID exists
        deleteForms(classId)
        locList.remove(classId)
        collectionAccounts.update_one(
            {"_id": ObjectId(flask.session.get('login'))},
            {"$set": {"classList":locList}}
        )
        collectionClassDB.remove({"_id":ObjectId(classId)})
        d = "removed"
    elif setting == "setPriorities":
        collectionClassDB.update_one(
            {"_id": ObjectId(flask.session.get('login'))},
            {"$set": {"qPriority":priorityList}}
        )
        d = "priority updated"
    else:
        d =" wat"
    return d

def deleteForms(classId):
    print classId
    formList = collectionClassDB.find_one({"_id":ObjectId(classId)}).get("formList")
    for fId in formList:
        collectionFormsDB.remove({"_id":ObjectId(fId)})
    return
##
#    settings
#        addForm
#            get parentId
#            get dictResponse
#            return "added"
#        getPriorities
#            get parentId
#            return "priorities gathered"

@app.route("/_formSettings")
def preForms():
    aReturn = request.args.get('aThing',0,type=str)
    aVal = json.loads(aReturn)
    setting = aVal.get("setting")
    parentId = aVal.get("parentId")
    dictResponse = aVal.get("dictResponse")
    d = formSettings(setting,parentId,dictResponse)
    #d = "yes"
    return jsonify(result = d)

def formSettings(setting,parentId,dictResponse):
    if setting == "addForm":
        aTime = arrow.utcnow().naive
        record = {"parentId":parentId,"dictResponse":dictResponse, "date":aTime,"teamNum": 0}
        print record
        collectionFormsDB.insert(record)
        aForm = collectionFormsDB.find_one({"date": aTime})
        locId = str(aForm.get('_id'))
        aClass = collectionClassDB.find_one({"_id":ObjectId(parentId)})
        locList = aClass.get("formList")
        locList.append(locId)
        collectionClassDB.update_one(
            {"_id": ObjectId(parentId)},
            {"$set": {"formList":locList}}
        )
        flask.session['priorityList'] = None
        d = "added"
    elif setting == "getPriorities":
        aClass = collectionClassDB.find_one({"_id":ObjectId(parentId)})
        aList = aClass.get('qPriority')
        print aList
        flask.session['priorityList'] = aList
        d = "true"
    else:
        d = "wat"
    return d

###############################################################################
###############################################################################

def funcTest(aClass,priority):
    return initGraph(aClass,1)

def scoreByPref(aClass,priority):
    locGraph = initGraph(aClass,0.0)
    aHash = getHash(aClass)
    aListOfNames = []
    for x in range(0,len(aHash)):
        aListOfNames.append(aHash[x].get('dictResponse').get('1'))
    for y in range(0,len(aHash)):
        for z in range(0,len(aListOfNames)):
            if aHash[y].get('dictResponse').get('10') == aListOfNames[z]:
                locGraph[y][z] = (2*priority)
    print locGraph
    return locGraph

def scoreByTime(aClass,priority):
    locGraph = initGraph(aClass,0.0)
    aHash = getHash(aClass)
    playerTimes = []
    for x in range(0,len(aHash)):
        playerTimes.append(aHash[x].get('dictResponse').get('4'))
    for y in range(0,len(playerTimes)):
        if playerTimes[y] == None:
            continue
        for z in range(0,len(playerTimes)):
            if playerTimes[z] == None:
                continue
            for w in range(0,len(playerTimes[y])):
                if y == z:
                    continue
                if playerTimes[y][w] in playerTimes[z]:
                    locGraph[y][z]+=(1*priority)
    print locGraph
    return locGraph


funcList = []
funcList.append(funcTest)
funcList.append(scoreByPref)
funcList.append(scoreByTime)
#funcList.append(preScoreByGrades)

def createTeams(classId,groupSizeMax):
    #initialize graph
    aClass = convert_class_id(classId)
    compGraph = initGraph(aClass,0.0)
    #loop through questions
    #for x in range(0,len(aClass.get('qPriority'))):
    for x in range(0,3):
        if aClass.get('qPriority')[x] == 0:
            continue
        else:
            addVals = funcList[x](aClass,1)
            compGraph = addGraphNums(compGraph,addVals)
    return getTeams(aClass,compGraph,groupSizeMax)

def initGraph(aClass,weight):
    compGraph = []
    for x in range(0,len(aClass.get('formList'))):
        innerList = []
        for y in range(0,len(aClass.get('formList'))):
            innerList.append(weight)
        compGraph.append(innerList)
    return compGraph

def addGraphNums(graphOne,graphTwo):
    for row in range (len(graphOne)):
        for col in range(len(graphOne)):
            graphOne[row][col] = (graphOne[row][col] + graphTwo[row][col])
    return graphOne

def getTeams(aClass,compGraph,groupSizeMax):
    currTeamScore = -1.0
    currTeam = []
    hashOfForms = getHash(aClass)
    for x in range(0,100000):
        randomTeam = getRandomGroup(hashOfForms,groupSizeMax,2.0)
        #randomTeam = getRandomGroup(hashOfForms,groupSizeMax)
        theScore = getGroupsScore(randomTeam,compGraph)
        if theScore > currTeamScore:
            currTeamScore = theScore
            print "Changed!"
            print currTeamScore
            currTeam = randomTeam
    groupsWithNames = convertNumsToNames(currTeam,hashOfForms)
    print currTeamScore
    return groupsWithNames

def getGroupsScore(groups,scoreMatrix):
    score = 0
    for g in groups:
        for x in range(0,len(g)):
            for y in range(0,len(g)):
                if x == y:
                    continue
                else:
                    score+=(scoreMatrix[g[x]][g[y]])
    #print score
    return score
    """"""""""
    #bogosort
    for g in groups:
        outerSol = 0.0
        count = 0
        #print g

        for val in g:
            if val != None:
                count+=1
        if count < 2:
            continue

        for x in range(0,len(g)):
            innerSol = 1.0
            for y in range(0,len(g)):
                if x == y or g[x] == None or g[y] == None:
                    continue
                else:
                    innerSol*=(scoreMatrix[g[x]][g[y]])
            #print innerSol
            outerSol+=(math.pow(innerSol,1/len(g)-1))
        #print outerSol
        score+=(outerSol/len(g))
    score /= len(groups)
    print score
    return score
    """""""""

def getRandomGroup(hashOfForms,groupSizeMax,minGroup):
#def getRandomGroup(hashOfForms,minGroup):
    tempList = []
    teams = []
    for x in range(0,len(hashOfForms)):
        tempList.append(x)
    random.shuffle(tempList)
    numGroups = int(len(tempList)/minGroup)
    for x in range(0,numGroups):
        teams.append([])
    overLoadCount = int(len(tempList) % minGroup)
    for x in range(0,overLoadCount):
        teams[x].append(tempList.pop())
    for x in range(0,int(minGroup)):
        for y in range(0,numGroups):
            teams[y].append(tempList.pop())
    #print teams
    """"""""""
    #bogosort
    while len(tempList) > 0:
        aTeam = []
        count = 0
        while len(aTeam) != groupSizeMax:
            if len(tempList) == 0:
                break
            av = random.randint(0,5)
            if  av == 1:
                aTeam.append(None)
                count+=1
            else:
                aTeam.append(tempList.pop(random.randint(0,len(tempList)-1)))
        if count != groupSizeMax:
            teams.append(aTeam)
    #print convertNumsToNames(teams,hashOfForms)
    """""""""
    return teams

def convertNumsToNames(teams,forms):
    teamNames = []
    for aTeam in teams:
        locTeam = []
        for person in aTeam:
            if person != None:
                locTeam.append(forms[person].get('dictResponse').get('1'))
        teamNames.append(locTeam)
    return teamNames

def getHash(aClass):
    aHash = []
    formList = aClass.get('formList')
    for x in range(0,len(formList)):
        aHash.append(convert_form_id(formList[x]))
    return aHash

##############################
##########Filters#############
##############################
@app.template_filter('convert_time')
def convert_time(timestamp):
    val = arrow.get(timestamp)
    return val.format('h:mm A')

@app.template_filter('classObj')
def convert_class_id(classId):
    val = collectionClassDB.find_one({"_id":ObjectId(classId)})
    return val

@app.template_filter('getPriorList')
def get_priorityList(classId):
    val = collectionClassDB.find_one({"_id":ObjectId(classId)}).get("qPriority")
    return val

@app.template_filter('formInfo')
def convert_form_id(formId):
    val = collectionFormsDB.find_one({"_id":ObjectId(formId)})
    return val

if __name__ == "__main__":
    import uuid
    app.secret_key = str(uuid.uuid4())
    app.debug = CONFIG.DEBUG
    app.logger.setLevel(logging.DEBUG)
    app.run(port=CONFIG.PORT,threaded=True)