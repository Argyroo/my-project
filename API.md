Προδιαγραφή Τυποποιημένων Διεπαφών REST (REST API Specification)

Τίτλος Έργου: Έξυπνοι Ανελκυστήρες — Σύστημα Προεπιλογής Προορισμού (Destination Dispatch)

Έκδοση Προδιαγραφής: 1.1.0

Ημερομηνία Έκδοσης: Νοέμβριος 2026

Πρωτόκολλο Επικοινωνίας: HTTP/1.1 REST

Μορφότυπος Δεδομένων: application/json; charset=utf-8

1. Επισκόπηση Αρχιτεκτονικής & Διαλειτουργικότητας

Το σύστημα βασίζεται στο μοντέλο Destination Dispatch (προεπιλογή ορόφου προορισμού πριν από την επιβίβαση) και αποτελείται από 4 αυτόνομα κατανεμημένα υποσυστήματα που εκτελούνται ως ανεξάρτητα CLI προγράμματα. Η επικοινωνία μεταξύ τους είναι πλήρως αποσυνδεδεμένη (decoupled) και βασίζεται αποκλειστικά σε τυποποιημένες κλήσεις REST API.

2. Ρόλοι Υποσυστημάτων:

 Α. Κεντρικό Σύστημα Ελέγχου (Central Controller):

Λειτουργεί ως HTTP Server για τα αιτήματα των επιβατών και των δοκιμών.

Λειτουργεί ως HTTP Client στέλνοντας εντολές προς τους κατά τόπους ανελκυστήρες.

Υλοποιεί τον ευρετικό/βελτιστοποιητικό αλγόριθμο ανάθεσης θαλάμων.

 Β. Διεπαφή Χρήστη (User Interface):

HTTP Client (τερματικό ορόφου). Συλλέγει τον επιθυμητό όροφο και εμφανίζει στον χρήστη το γράμμα του ανατεθειμένου ανελκυστήρα.

 C. Σύστημα Ελέγχου Ανελκυστήρα (Elevator Controller):

HTTP Server για κάθε θάλαμο (A, B, C...). Δέχεται νέες στάσεις και εκθέτει σε πραγματικό χρόνο τηλεμετρία (θέση, ταχύτητα, κατάσταση θυρών, επιβάτες).

 D. Σύστημα Δοκιμών και Αξιολόγησης (Evaluation System):

HTTP Client που πυροδοτεί σενάρια μαζικών κλήσεων, μετρά χρόνους αναμονής/διαδρομής και συγκρίνει επιδόσεις.
Μέθοδος      Endpoint                            Server (Παροχέας)    Client (Καλεστής)  

POST        /api/v1/dispatch                     Central Controller  User Interface / Evaluator

POST       /api/v1/elevators/{id}/destinations   Elevator Controller   Central Controller

GET        /api/v1/elevators/{id}/status         Elevator Controller   Central / Evaluator

GET       /api/v1/system/overview                Central Controller     Evaluation System

POST     /api/v1/evaluation/run             Evaluation System / Central  Evaluation System


3. Αναλυτικές Προδιαγραφές Διεπαφών (API Endpoints)

3.1 Αίτημα Επιβάτη (Destination Dispatch)

Διαδρομή: /api/v1/dispatch

Μέθοδος: POST

Σκοπός: Το τερματικό ορόφου στέλνει την αφετηρία και τον προορισμό του επιβάτη. Το κεντρικό σύστημα αποφασίζει και επιστρέφει άμεσα τον ανελκυστήρα.


α. Request Payload Schema & Example:

{
  "sourceFloor": 0,
  "destinationFloor": 14,
  "passengerCount": 1
}


sourceFloor (integer, υποχρεωτικό): Ο όροφος στον οποίο βρίσκεται ο επιβάτης (3 έως 200).

destinationFloor (integer, υποχρεωτικό): Ο επιθυμητός όροφος αποβίβασης.

passengerCount (integer, προαιρετικό, προεπιλογή: 1): Πλήθος ατόμων της κλήσης.

β. Success Response (HTTP 200 OK):

{
  "status": "ASSIGNED",
  "assignedElevator": "B",
  "sourceFloor": 0,
  "destinationFloor": 14,
  "estimatedWaitSeconds": 18,
  "requestId": "req-9843-af7"
}


assignedElevator (string): Το μοναδικό αναγνωριστικό γράμμα του επιλεγμένου ανελκυστήρα (A, B, C...).

estimatedWaitSeconds (integer): Εκτιμώμενος χρόνος μέχρι την άφιξη του θαλάμου στον όροφο επιβίβασης.

γ. Error Responses:

HTTP 400 Bad Request: Μη έγκυρος όροφος ή ίδιος όροφος αφετηρίας/προορισμού.

{
  "errorCode": "INVALID_FLOOR_RANGE",
  "message": "Ο όροφος προορισμού (250) βρίσκεται εκτός του εύρους λειτουργίας του κτιρίου (0 - 200)."
}

HTTP 503 Service Unavailable: Κανένας ανελκυστήρας δεν είναι σε λειτουργική κατάσταση.


3.2 Εντολή Στάσης προς Ανελκυστήρα

Διαδρομή: /api/v1/elevators/{id}/destinations

Μέθοδος: POST

Σκοπός: Το κεντρικό σύστημα αναθέτει στάση στον συγκεκριμένο θάλαμο ({id} = αναγνωριστικό γράμμα, π.χ. B).

Request Parameters:

Path Parameter {id}: Το διακριτικό του ανελκυστήρα (π.χ. B).

Request Payload:

{
  "stopFloor": 14,
  "stopReason": "DROPOFF"
}


stopFloor (integer, υποχρεωτικό): Ο όροφος που πρέπει να προστεθεί στη διαδρομή.

stopReason (string, προαιρετικό): "PICKUP", "DROPOFF", "MAINTENANCE".

Success Response (HTTP 200 OK):

{
  "elevatorId": "B",
  "status": "ACCEPTED",
  "currentFloor": 4,
  "targetFloor": 14,
  "activeRoute": [8, 14]
}


Error Response:

HTTP 404 Not Found: Δεν υπάρχει ανελκυστήρας με το συγκεκριμένο ID.

{
  "errorCode": "ELEVATOR_NOT_FOUND",
  "message": "Ο ανελκυστήρας 'Z' δεν βρέθηκε στο σύστημα."
}


3.3 Κατάσταση & Τηλεμετρία Ανελκυστήρα

Διαδρομή: /api/v1/elevators/{id}/status

Μέθοδος: GET

Σκοπός: Επιστρέφει την τρέχουσα φυσική και λειτουργική κατάσταση ενός ανελκυστήρα.

Success Response (HTTP 200 OK):

{
  "elevatorId": "B",
  "currentFloor": 6,
  "targetFloor": 8,
  "direction": "UP",
  "doorState": "CLOSED",
  "motionState": "MOVING",
  "currentLoadKg": 210.0,
  "capacityKg": 800.0,
  "passengerCount": 3,
  "activeRoute": [8, 14],
  "operationalStatus": "OK"
}


direction (string): "UP", "DOWN", ή "IDLE".

doorState (string): "OPEN", "CLOSED", "OPENING", "CLOSING".

motionState (string): "MOVING", "STOPPED", "DECELERATING".

operationalStatus (string): "OK", "OVERLOAD", "FAULT", "OUT_OF_SERVICE".

3.4 Συγκεντρωτική Επισκόπηση Συστήματος

Διαδρομή: /api/v1/system/overview

Μέθοδος: GET

Σκοπός: Επιστρέφει συνοπτική εικόνα όλων των ανελκυστήρων για χρήση από την οθόνη επόπτη ή το σύστημα αξιολόγησης.

Success Response (HTTP 200 OK):

{
  "timestamp": 1730718000,
  "totalElevators": 3,
  "elevators": [
    {
      "elevatorId": "A",
      "currentFloor": 0,
      "direction": "IDLE",
      "doorState": "CLOSED",
      "passengerCount": 0
    },
    {
      "elevatorId": "B",
      "currentFloor": 6,
      "direction": "UP",
      "doorState": "CLOSED",
      "passengerCount": 3
    },
    {
      "elevatorId": "C",
      "currentFloor": 12,
      "direction": "DOWN",
      "doorState": "OPEN",
      "passengerCount": 2
    }
  ]
}


3.5 Εκτέλεση Σεναρίου Αξιολόγησης (Benchmark Run)

Διαδρομή: /api/v1/evaluation/run

Μέθοδος: POST

Σκοπός: Εκκινεί αυτόματα εκτέλεση προσομοίωσης κυκλοφορίας με παραμέτρους για τη συγκριτική αξιολόγηση των αλγορίθμων.

Request Payload:

{
  "scenarioName": "MORNING_UP_PEAK",
  "durationSeconds": 180,
  "passengerArrivalRatePerMin": 25,
  "buildingConfig": {
    "floors": 15,
    "elevators": 4
  }
}


Success Response (HTTP 200 OK):

{
  "benchmarkId": "bench-2026-11-04-01",
  "status": "COMPLETED",
  "metrics": {
    "totalPassengersServed": 75,
    "averageWaitTimeSeconds": 14.8,
    "maxWaitTimeSeconds": 41.2,
    "averageTransitTimeSeconds": 32.5,
    "energyConsumptionScore": 88.4
  }
}


4. Κώδικες Κατάστασης HTTP (HTTP Status Codes)

Όλα τα υποσυστήματα οφείλουν να ακολουθούν πιστά τους ακόλουθους κωδικούς HTTP για τη διασφάλιση συμβατότητας μεταξύ διαφορετικών ομάδων:

200 OK: Το αίτημα εκτελέστηκε επιτυχώς και επιστρέφεται το αναμενόμενο JSON payload.

202 Accepted: Το αίτημα έγινε δεκτό για ασύγχρονη επεξεργασία (π.χ. έναρξη σεναρίου δοκιμών).

400 Bad Request: Συντακτικό λάθος στο JSON ή μη έγκυρες παράμετροι εισόδου (π.χ. sourceFloor == destinationFloor).

404 Not Found: Το ζητούμενο resource ή αναγνωριστικό ανελκυστήρα δεν υφίσταται.

422 Unprocessable Entity: Το JSON είναι συντακτικά ορθό, αλλά η επιχειρησιακή λογική απορρίπτει την ενέργεια (π.χ. υπέρβαρο θαλάμου).

500 Internal Server Error: Απρόβλεπτο σφάλμα κατά την εκτέλεση στον server.

503 Service Unavailable: Προσωρινή αδυναμία εξυπηρέτησης (π.χ. απώλεια επικοινωνίας με όλους τους ανελκυστήρες).

5. Οδηγός Ανεξάρτητης Δοκιμής (cURL Examples)

Κάθε ομάδα μπορεί να ελέγξει τις διεπαφές της από τη γραμμή εντολών:

Δοκιμή κλήσης επιβάτη:

curl -i -X POST http://localhost:8080/api/v1/dispatch \
  -H "Content-Type: application/json" \
  -d '{"sourceFloor": 0, "destinationFloor": 10}'


Έλεγχος κατάστασης ανελκυστήρα B:

curl -i -X GET http://localhost:8082/api/v1/elevators/B/status


Προσθήκη στάσης στον ανελκυστήρα B:

curl -i -X POST http://localhost:8082/api/v1/elevators/B/destinations \
  -H "Content-Type: application/json" \
  -d '{"stopFloor": 10}'
