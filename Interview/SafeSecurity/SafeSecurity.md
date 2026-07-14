2nd interview after first screening round:

Was asked some basic questions like:
1. Diff between http and https. I gave the answer that http is kind of like a language/protocol through which the systems communicate and if we add security to mitigate man in the middle attacks then SSL gets involved) which appends the S at the end of https.
2. Diff between sql and no sql. I wasn't able to give a proper answer. I gave the answer like sql has fixed query structure and fixed schema (to create a column you will have to add it explictly through query but not the case with no sql) in no sql you can just directly add data and it would support that.
3. What is cap theorem and how does it apply to master slave architecture (is it AP or CP system). I gave the answer that it depends as to if the master is syncing with slaves before sending OK confirmation then CP otherwise AP.
4. Diff between semaphores and mutex. I gave the answer as semaphores multiples candies and mutex single candy
5. What happens in the browser when you do: google.com
	My answer was, not sure how correct it is and I have missed a lot of things:
	a. browser cache 
	b. OS cache
	c. Internet service provider
	d. Now here I had directly gone to authoritative name server like (.com, .in, .gov)
6. If there is only one core and there is only one process running on it so do we still need resource locking? -> I gave the answer in terms if there are multiple threads in the process then yes otherwise no.
7. What is a load balancer. I gave the answer that load balancers can distribute load and saves a particular service from going down in case of traffic coming from mumbai to the nearest server. 
8. How TCP connection is established. I mentioned that the client sends 'clien hello' the server responds with yes I can connect along with the tls versions it supports and then the client sees the common versions amongst themselves and sends the response back to the server to say that we can connect on these algos.
9. Difference between TCP and UDP. I mentioned that TCP guarantees 2 things like order to be maintained on the server/client side of the packets and make sure that the client is getting that call delivered successfully (my questions is: but how does it do that?).
10. What will happen in case of extensive indexing on the database.
11. Design a LRU cache and in that give the functionality for these two functions get() and put():
In this I missed just the one thing:
Removing from the map and in order to do that we need to store they key of the map as well in the node cause otherwise how will we find the key to be deleted from the map.

Map -> 
Need to maintain order as well -> Queue

Update -> Doubly linked list
Removal -> Cache exceeds the limit


// HashMap cacheMap
// DLL cacheOrder
// start -> <- element -> <- end // THIS IS THE IMPORTANT THING HERE.. I'M USING 2 DUMMY NODES START AND END.
// Node // IMPPPPP TO ALSO HAVE THE KEY SAVED IN THE NODE AS WE WILL HAVE TO DELETE THE KEY FROM THE QUEUE WHEN IT OVERFILLS.
// maxSize

int get(int key) {
    if (cacheMap.containsKey(key)) {

        // Removing it from current place/order.
        Node node = cacheMap.get(key);
        Node nextNode = node.next;
        Node prevNode = node.prev;

        node.next = null;
        node.prev = null;
        nextNode.prev = prevNode;
        prevNode.next = nextNode;

        // Inserting the node at the beginning
        Node firstNode = start.next;
        start.next = node;
        firstNode.prev = node;
        node.prev = start;
        node.next = firstNode;

        return node.val;
    }

    return 0;
}

void put(int key, int value) {

    // What if key is existing
    // What if key is absent

    if (cacheMap.containsKey(key)) {

        get(key);
        cacheMap.get(key).val = value;

    } else {

        if (maxSize > cacheMap.size()) {

            Node node = new Node(value);

            Node firstNode = start.next;
            start.next = node;
            firstNode.prev = node;
            node.prev = start;
            node.next = firstNode;

            cacheMap.put(key, node);

        } else {

            // Removing the val
            Node lastNode = end.prev;
            Node newLastNode = lastNode.prev;

            newLastNode.next = end;
            end.prev = newLastNode;
            lastNode.prev = null;
            lastNode.next = null;

			// Here I missed a very big thing.. Removing from the map and in order to do that we need to store they key of the map as well in the node cause otherwise how will we find the key to be deleted from the map.

            //
            Node node = new Node(value);

            Node firstNode = start.next;
            start.next = node;
            firstNode.prev = node;
            node.prev = start;
            node.next = firstNode;

            cacheMap.put(key, node);
        }
    }
}

Interview 2:

Was asked to share the HLD of URL shortner. I started with clearing the requirements but the interviewer was not very friendly, was kind of not interested in taking the interview itself. I started designing the system itself and got deep in fixing the small issues. He wanted me to explain in C4 model (which I didn't know before). Was rejected in this round.

The interviewer after starting with my C4 design, wanted me to elaborate each and every aspect and then he had a question as to how we could make that a SAAS product.

Places where I faced problems were in calculating the load on the system quickly and efficiently, I didn't know if a character was 1 byte or not. I'll be attaching the draw.io file in this folder for the same. I also mentioned that the shortened URL will be calculated based on the hash generated by md5 (didn't remember any prominent algo's at the time), and didn't consider what if 2 different URL then hash to the same value.

Generally incremented ids are being used here or range based id assignment to microservices, or we can use snowflake as well.

The reason we're using an incremented id instead of using a hashing function is that I would want to see analytics for my generated URL like suppose if how many people clicked on my google.com shortened URL then I would not be able to decipher that so its recommened to give each user its own short URL cause of that.
