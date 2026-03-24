## Introduction:
I spent a considerable amount of time configuring the environment, as I do not have access to Linux or macOS. The provided scripts were not compatible with my setup. I attempted to run them using WSL and within a Docker container, but without success. Ultimately, I decided to deploy an EC2 Workstation as outlined in the USER_MANUAL. However, executing the required script also failed on Windows. I then set up Ubuntu on VMware, ran the scripts there, and only after that was I finally able to get the environment fully operational.

## Assignment 1
The results of Assignment 1 (deploying all environments) are available in `./assignment-1-endpoints.txt`. All scripts ran successfully without any issues.

## Assignment 3

query_time_ms:
- EC2: 24.668 ms
- Fargate: 27.585 ms
- Lambda ZIP: 63.611 ms
- Lambdad Container: 27.628 ms

Due to unexpected errors in EC2 responses, I reduced concurrency for Fargate and EC2 to 40. 

Environment	            Concurrency     p50 (ms)    p95 (ms)    p99 (ms)    Server avg (ms)
Lambda (zip)	            5				15.6        23.1        67.7        16.0
Lambda (zip)	            10				11.7        19.4        66.7        13.5
Lambda (container)	        5				15.5        21.7        317.8       18.3
Lambda (container)	        10				11.7        21.0        67.1        13.7
Fargate	                    10				903         1215        1388        916
Fargate	                    40				3702        4082        4216        3593
EC2	                        10				201.1       260.9       294.2       202.3
EC2	                        40				760         949         1024        748

Question: Annotate any cell where p99 > 2× p95 (this signals tail latency instability).
p99 > 2 × p95 is true for: Lambda (container), Lambda (zip).

Question: Explain why Lambda p50 barely changes between c=5 and c=10 (each request gets its own execution environment), while Fargate/EC2 p50 increases significantly between c=10 and c=50 (requests queue on a single task/instance).
The difference in p50 latency between AWS Lambda and Fargate/EC2 comes from how each platform handles multiple requests and resources. In AWS Lambda, each request gets its own separate environment with its own CPU and memory, so it doesn’t have to wait. Lambda creates one execution environment per concurrent request. In Fargate and EC2, all requests share the same instance or task, so when many requests come at once, they compete for resources and may wait in a queue, which makes p50 latency higher.

Question: Explain what causes the latency difference between server-side query_time_ms and client-side p50.
As shown above, query_time_ms is much shorter than p50. That is because the server-side query_time_ms measures only the execution of the task, while the client-side measurement contains: Network RTT, TLS handshake, ALB overhead, DNS resolution, initialization, envrionment overhead, queeuing and much more.

## Assignment 4
For this assignment, I used `loadtest/scenario-c.sh`.

Environment	            Concurrency     p50 (ms)    p95 (ms)    p99 (ms)    Server avg (ms)
Lambda (zip)	            10				17.5        119.8       124.5       22.6
Lambda (container)	        10				16.3        123.7       129.3       21.3
Fargate	                    40				4678        5287        5507        4271
EC2	                        40				694         891         941         682

In Scenario C - Burst from Zero, Lambda (zip and container) handles the burst efficiently because each request gets its own isolated execution environment, so the median latency stays very low (~17 ms) and only a small portion of requests experience slightly higher delays (~120–130 ms) due to cold starts after idling. Fargate shows very high latencies (p50 ~4.7 s, p99 ~5.5 s) because all requests share the same tasks and the platform must start new containers under sudden load, causing queuing and resource contention. EC2 has moderate latencies (p50 ~694 ms, p99 ~941 ms) because while it can handle multiple requests on the same instance, the CPU and memory are shared, so some requests wait in line, making it slower than Lambda but faster than Fargate during the burst.

Question: Explain why Lambda's burst p99 is much higher than Fargate/EC2.
Actually, looking at the raw statistics, Lambdas p99 is much smaller than Fargate and EC2 p99. However, considering the variance and the difference between p50 and p99 for Lambda, it becomes clear that cold starts still affect some requests, causing them to take significantly longer.

Question: Identify the bimodal distribution in Lambda latencies (warm cluster vs. cold-start cluster).
It is clear that during cold starts some requests take significantly longer, which is reflected in the p99 value. During a cold start, the p99 latency is around 125 ms, whereas for warm starts it is much lower, approximately 67 ms.

Question: State whether Lambda meets the p99 < 500ms SLO under burst. If not, explain what would need to change.
Yes, Lambda meets the p99 < 500ms SLO under burst from zero.

## Assignment 5

[EC2 Cost](figures/ec2_cost.png)
[Fargate Cost](figures/fargate_cost.png)
[Lambda Cost](figures/lambda_cost.png)

Compute monthly idle cost assuming 18 hours/day idle, 6 hours/day active.


EC2:
On EC2, it doesn't matter whether idle or active.
Monthly cost: $0.0208 × 24 × 30 = $14.98

Lambda:
On Lambda, we pay only for number of requests and duration of the request.
Thus, we don't pay for idle time
Monthly cost: $0
Unless Provisioned Concurrency is enabled, then we pay $0.0000041667 for every GB-second.

Fargate:
On Fargate, it doesn't matter whether idle or active. 
We have: 0.5 vCPU, 1 GB
vCPU cost:   0.5 × $0.04048 × 24 × 30 = $14.57
Memory cost: 1.0 × $0.004445 × 24 × 30 = $3.20
Monthly cost: $17.77/month

## Assignment 6
