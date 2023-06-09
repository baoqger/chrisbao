提纲:

结合项目介绍: 各个service都是什么, 为什么选择了对应的service 类型

技术细节介绍with code

: communication perspective:  create actor with RPC and HTTP request
: naming perspective
: replication perspective: 
: partitions and replicas of stateful service
: reliable collections of stateful services

延展讨论:

sf vs k8s 

服务通信: RPC, message passing, async of message broker(event-driven application), internal DNS

scalability and availability:  instance(actor service), partition(county service) and replicas

cloud native advantages

explore the techniques under the hood: NAT, simulated annealing, Paxos consensus algorithm 



展开讨论: 

service fabric的问题, service mesh的优势