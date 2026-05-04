## Project Distributed system alert.
 
- Client (persons) register themselves to a specific group via a server.
- General security person plays the role of administrator and we will be able to send alerts to all the registered persons belonging to his group (or groups)
``Remark: many security persons can be available but the groups are disjoint between a security person and another. Only security persons send alert.``
- Persons can leave questions to the security person group that should be kept in his/her inbox to be able to optionally reply to them later.
- For simplification, anytime the user has to enter the system, he should be only identified by a unique username (chooses a username and participates with a group and he will be able to be identified…) no need for DB, use text file for keeping info like credential or msg.
- Use 1 either TCP or UDP the easiest is the best.
- Any additional steps are welcome.
