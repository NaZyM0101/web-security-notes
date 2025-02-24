​        Perhaps the most intuitive form of these race conditions are  those that involve sending requests to multiple endpoints at the same  time.    

​        Think about the classic logic flaw in online stores where you  add an item to your basket or cart, pay for it, then add more items to  the cart before force-browsing to the order confirmation page.    

#### Note

​            If you're not familiar with this exploit, check out the  Insufficient workflow validation lab from our Business logic  vulnerabilities topic.        

​        A variation of this vulnerability can occur when payment  validation and order confirmation are performed during the processing of a single request. The state machine for the order status might look  something like this:    

![Race variation of the basket adjustment logic flaw](./images/race-condition/race-conditions-basket-adjustment-race.png)

​        In this case, you can potentially add more items to your basket  during the race window between when the payment is validated and when  the order is finally confirmed.    

### Aligning multi-endpoint race windows

​        When testing for multi-endpoint race conditions, you may  encounter issues trying to line up the race windows for each request,  even if you send them all at exactly the same time using the  single-packet technique.    

![Aligning multi-endpoint race windows](./images/race-condition/race-conditions-multi-endpoint.png)

​        This common problem is primarily caused by the following two factors:    

- **Delays introduced by network architecture -** For example, there may be a delay whenever the front-end server  establishes a new connection to the back-end. The protocol used can also have a major impact.
- **Delays introduced by endpoint-specific processing -** Different endpoints inherently vary in their processing times,  sometimes significantly so, depending on what operations they trigger.

​        Fortunately, there are potential workarounds to both of these issues.    

#### Connection warming

​        Back-end connection delays don't usually interfere with race  condition attacks because they typically delay parallel requests  equally, so the requests stay in sync.    

​        It's essential to be able to distinguish these delays from those caused by endpoint-specific factors. One way to do this is by "warming" the connection with one or more inconsequential requests to see if this smoothes out the remaining processing times. In Burp Repeater, you can  try adding a `GET` request for the homepage to the start of your tab group, then using the **Send group in sequence (single connection)** option.    

​        If the first request still has a longer processing time, but the rest of the requests are now processed within a short window, you can  ignore the apparent delay and continue testing as normal.    

​        If you still see inconsistent response times on a single  endpoint, even when using the single-packet technique, this is an  indication that the back-end delay is interfering with your attack. You  may be able to work around this by using Turbo Intruder to send some  connection warming requests before following up with your main attack  requests.    

#### Abusing rate or resource limits

​        If connection warming doesn't make any difference, there are various solutions to this problem.    

​        Using Turbo Intruder, you can introduce a short client-side  delay. However, as this involves splitting your actual attack requests  across multiple TCP packets, you won't be able to use the single-packet  attack technique. As a result, on high-jitter targets, the attack is  unlikely to work reliably regardless of what delay you set.    

![Introducing a client-side delay between requests](./images/race-condition/race-conditions-client-side-delay.png)

​        Instead, you may be able to solve this problem by abusing a common security feature.    

​        Web servers often delay the processing of requests if too many  are sent too quickly. By sending a large number of dummy requests to  intentionally trigger the rate or resource limit, you may be able to  cause a suitable server-side delay. This makes the single-packet attack  viable even when delayed execution is required.    

![Abusing rate or resource limits to introduce a server-side delay](./images/race-condition/race-conditions-abusing-rate-or-resource-limits.png)