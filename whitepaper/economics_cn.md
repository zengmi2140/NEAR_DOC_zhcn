# math in the easay

1
$$
epochReward_t = coinbaseReward_t + epochFees_t 
\tag{1}$$  

2  
$$
maxCoinbase = totalSupply_t \times (\sqrt[numEpochsPerYear]{1+maxInflation} - 1) 
\tag{2}$$  

3  
$$
coinbaseReward_t = 
\begin{cases}
0 & epockFees_t \geq maxCoinbase \\
maxCoinbase-epochFees_t & otherwise
\end{cases} 
\tag{3}$$

4
$$
epochFee_t = \sum_{i=firstBlock_t}^{lastBlock_t}(1-devoloperPct) \times txFee_i + stateFee_i 
\tag{4}$$

5
$$
gas_{tx} = numberOfCPUInstructions(tx) + \alpha \times SizeOf(tx) 
\tag{5}$$

6
$$
seatPrice = \argmin_{x\in V} \bigg ( \sum_{v\in V} \lfloor \frac{stake_v}{x} \rfloor \geq numSeats \bigg) 
\tag{6}$$

7
$$
totalValidatorReward_t = (1-prtocolPct) \times (coinbaseRewards_t + epochFees_t) 
\tag{7}$$

8
$$
validator_vReward_t = uptime_t^v \times totalValidatorReward_t \times numSeats_v 
\tag{8}$$

9
$$
uptime_t^v = 
\begin{cases}
0 & uptimePct_t^v \lt onlineThreshold \\
\frac{uptimePct_t^v-onlineThreshold}{1-onlineThreshold} & otherwise
\end{cases} 
\tag{9}$$

10
$$
developerReward_{index}^{account} = developerPct \times txFee_{index}^{account}
\tag{10}$$

11
$$
protocolReward_t = protocolPct \times (coinbaaseReward_t + epochFee_t)
\tag{11}
$$