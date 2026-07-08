create an api-led connectivity network example with the 3 layers (experience, process, system)

create a customers + orders architecture for this.

these are the operations you can do:
- list all the customer
- list one customer's details
- get each customer's orders
- see one customer's order's details
- create a new customer
- edit an existing customer's details
- delete a customer only if no orders attached
- create a new order attached to a customer
- edit an existing order (still attached to a customer)
- cancel an existing order (this does not delete it)
- delete an existing order

we are not going to be connecting to an actual db just yet. mock and seed test data in the system layer as if we were connected to a db.

create as many apis as you think is needed for this architecture to be successful and to really follow api-led connectivity, without having too much redundancy that code is repeated but also dont just create one big api for everything. there must be AT LEAST 3 apis since there are 3 layers, but there can be more than 3 apis if you choose that. 

choose the best approach. think like a mulesoft architect. use the latest mule, maven, and dw version for the mule projects.