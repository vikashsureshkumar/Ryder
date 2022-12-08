# to build a code where low price ryde is assigned to the booking customer

# import statements to use inbuilt libraries
import mysql.connector
from mysql.connector import Error
from math import cos, asin, sqrt, pi
import random
# import statement ends
# global cursor defined to use it in all functions instead of passing
mycursor = None


# function to get mysql connection
def getConnection():
    try:
        connection = mysql.connector.connect(host='localhost', database='ryder', user='root', password='Latha#123')
        if connection.is_connected():
            print("MySQL connection is opened")
            return connection
    except Error as e:
        print("Error while connecting to MySQL", e)
# returns connection


# function to close the connection
def closeConnection(connection):
    try:
        if connection.is_connected():
            connection.commit()
            connection.close()
            print("MySQL connection is closed")
    except Error as e:
        print("Error while closing connecting to MySQL", e)
# returns nothing


# function to get all available rydes which are requested in any platform or app
def getAllAvailableRydes():
    try:
        mycursor.execute("select * from rydes where status = 0")
        return mycursor.fetchall()
    except Exception as e:
        print("Exception Occurred", repr(e))
# returns gathered rydes data


# function to get all available drivers who are currently not assigned any trips
def getAllAvailableDrivers():
    try:
        mycursor.execute("select d.id, d.first_name, d.last_name, drl.location, drl.status from driver_realtime_location as drl join drivers as d on d.id = drl.driver_id where drl.status = 0")
        return mycursor.fetchall()
    except Exception as e:
        print("Exception Occurred", repr(e))
# returns gathered drivers data


# function to get distance between two latitude and longitude using haversine formula
def distance(lat1, lon1, lat2, lon2):
    try:
        lat1, lon1, lat2, lon2 = float(lat1), float(lon1), float(lat2), float(lon2)
        p = pi / 180
        a = 0.5 - cos((lat2 - lat1) * p) / 2 + cos(lat1 * p) * cos(lat2 * p) * (1 - cos((lon2 - lon1) * p)) / 2
        return 12742 * asin(sqrt(a))
    except Exception as e:
        print("Exception Occurred", repr(e))
# returns calculated data


# function to get distance between the pickup and dropoff location to calculate the fare and distance
def getDistanceForRides(rideData):
    try:
        for ryde in rideData:
            pickupLoc = str(ryde['pickup_location']).split(',')
            dropOffLoc = str(ryde['dropoff_location']).split(',')
            ryde['distance'] = round(distance(pickupLoc[0], pickupLoc[1], dropOffLoc[0], dropOffLoc[1]), 2)
            ryde['fare'] = int(round(ryde['distance'] * 3, 2))
        return rideData
    except Exception as e:
        print("Exception Occurred", repr(e))
# returns gathered distance and fare data


# function to get all eligible drivers within 5kms to be eligible for the trip
def eligibleDriverRydes(rydes, drivers):
    try:
        for ryde in rydes:
            pickupLoc = str(ryde['pickup_location']).split(',')
            for driver in drivers:
                driverLoc = str(driver['location']).split(',')
                distanceCalculated = distance(pickupLoc[0], pickupLoc[1], driverLoc[0], driverLoc[1])
                if distanceCalculated < 5:
                    try:
                        ryde['eligibleDrivers'].append(driver['id'])
                    except:
                        ryde['eligibleDrivers'] = [driver['id']]
            if "eligibleDrivers" not in ryde:
                ryde['eligibleDrivers'] = []
        return sorted(rydes, key=lambda r: r['fare'], reverse=True)
    except Exception as e:
        print("Exception Occurred", repr(e))
# returns gathered data for eligible rydes and drivers


# function to assign a driver to the trip based on eligibility
def assignRydersForRyde(rydeDetails):
    try:
        driverStatus = {}
        drivertripMap = []
        driverMap = []
        for ryde in rydeDetails:
            ryde["pickup_time"] = str(ryde["pickup_time"])
            ryde["estimated_ride_duration"] = str(ryde["estimated_ride_duration"])
            while True:
                if ryde['eligibleDrivers']:
                    assigningDriverId = random.choice(ryde['eligibleDrivers'])
                    if assigningDriverId in driverStatus and driverStatus[assigningDriverId] == 1:
                        ryde['eligibleDrivers'].remove(assigningDriverId)
                    else:
                        ryde['driver_id'] = assigningDriverId
                        driverStatus[ryde['driver_id']] = 1
                        ryde['eligibleDrivers'].remove(ryde['driver_id'])
                        drivertripMap.append((1, ryde['driver_id'], ryde['id']))
                        driverMap.append((1, ryde['driver_id']))
                        del ryde['eligibleDrivers']
                        break
                else:
                    del ryde['eligibleDrivers']
                    break

        if drivertripMap:
            mycursor.executemany("UPDATE rydes SET status = %s , driver_id = %s where id = %s", drivertripMap)
            mycursor.executemany("update driver_realtime_location set status = %s where driver_id = %s", driverMap)
        return rydeDetails
    except Exception as e:
        print("Exception Occurred", repr(e))
# returns gathered data and executes update query to assign trips


# function to gather and return data to main fucntion
def getRydeDetails():
    try:
        return assignRydersForRyde(eligibleDriverRydes(getDistanceForRides(getAllAvailableRydes()), getAllAvailableDrivers()), )
    except Exception as e:
        print("Exception Occurred", repr(e))
# returns gathered data


if __name__ == '__main__':
    # makes connection
    connection = getConnection()
    # creates cursor
    mycursor = connection.cursor(dictionary=True)
    # get data based on trips and drivers
    rydeDetails = getRydeDetails()
    # print details
    print("rydeDetails", rydeDetails)
    # close connection since all process is completed
    closeConnection(connection)

