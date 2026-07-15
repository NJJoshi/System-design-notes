
### Topic: Functional vs Non-Functional Requirement

- Get non-function requirements from interviewers by asking below common questions OR look for answers into given requirement for below mentioned questions:
    1. Expected Traffic
    2. Read/Write Ratio
    3. Latency Targets
    4. Availability needs
    5. Data consistency requirements
    6. Failure tolerance.

This signals senior level thinking while attempting system design interviews.

- Key Trade-off is, Simplicity Vs Gurantees.
- Every single choice of component into system has be strongly justified by you.

### Topic: Asking Clarifying Questions effectively
- Systen design questions are intentionally ambiguous to understand how candidate's capability to understand problem statement & it's simplification capability.
- Don't assume details and rush ahead to solve it.
- Even though we think problem statement is absolutly clear, please confirm your understand with interviewer by explaining your understanding.
- Ask questions if required. Good questions slow down problem in productive way.
- Ask questions at very begining and at point where we are making major decision while attempting system design.
- Trade - off here is momentum vs precision.
- If candidate's question leads to changing design direct that it will consider as good one otherwise it can be consider as waste of time.

### Topic: Capacity Estimation Using Rough Math
- This is needed for How big and how fast system needs to be
- The goal is not precision. The goal is order of magnitude in correct way.
 - If our guestimate numbers are off by 20% then its ok while guessing statics. Interviewers care about reasining is sound or not.
 - Use rough math in correct way after clarifying function & non-functional requirements.
 e.g.
 Total Users -> Actions per User -> Request per Seconds -> Peak Traffic -> Storage Growth
 - Always think into "per second" or "per day" numbers. Those maps directly to system limits.
 - Peak load matters more than averages. Always mention "Peak to Average Ratios" for that specific system design and design always for "Peak Load".
 - Here main trade-off is "speed vs accuracy". While achieving more accuracy takes more time in correction will be result into rejection because it will waste interviewers time. 
 - Also don't move too fast with assumptions. If we are making assumptions, we should be announcing it during interview and make adjustment into those assumption in interview in future if required.

 ### Topic: Golden Rules of System Design Interviews
 - There is no perfect system exists
 - Current solutions are calculated trade-off driven solutions available into System
 - Interviewer will like to see during system design interviews that why one component has been picked up with proper justication.
 - while picking up any tools like DB (e.g PostGresSQL, MySQL), Caching (e.g. Redis, Hazelcast, Memcache) etc with their pros-cons.
 - This turns design from opinion based to reasoning base.
 - Strong candidates constantly anchors decisions to requirement like latency over consistency, simplicity over flexibility, cost over scale.
 - Interviewers always prefer clarity over ambiguity for a specific design.