## Instruction Cache Prefetching on Pin icache
The instruction cache (icache.c) provided by Intel's PIN tool is modified by adding a prefetching function.
Serveral caching algorithms were implemented on the original instruction cache code and their performance were evaluated and compared.

### Instruction Cache Misses
Instruction cache misses happens simply when next the instructions to be fetched are not in the cache. This could could be fatal for high performance processors and superscalar architectures, as the entire system would be expotentially inefficient due to a few cache misses in a cycle.

### Cache prefetching 
A good cache prefetcher, which ideally eliminate all cache misses, should be able to: 
- predict when exactly the cache lines would soon be brought into the cache
- start prefetching just in time the instruction/data is needed.

Three methods of prefetching would be used and compared in this project: 
  - [Next line prefecthing](#next_line_prefetching)
  - [Target Line prefetching](#target_line_prefetching)
  - [Wrong path prefetching](#wrong_path_prefetching)
  
### Benchmarks
[FindPi.c](https://github.com/amandazhuyilan/Huntington-Ave-icache/blob/master/FindPi.c) and [Barnes.c](https://github.com/amandazhuyilan/Huntington-Ave-icache/blob/master/Barnes.c) would be used as benchmarking programs to test the instruction cache misses on the the prefecthing algorithms. FindPi.c computes the value by using the Monte-Carlo algorithm and Barnes.c uses the Barnes-Hut methods to simulate the interaction of a system of bodies in astronomy.

### Pin
Thanks to the perfect [documentation](https://software.intel.com/en-us/articles/pin-a-dynamic-binary-instrumentation-tool) that came with Intel's Pin, a dynamic binary instrumentation tool, it did not take me long to figure out the two main componenets in icache: instrumentation and analysis. 

Instrumentation decides where and what code is inserted at the location, it would be called to evaluate the properties of the code to be presented, while the analysis determines the code to be executed at the insertion point, collects information about the program to be analyzed. Next-line prefetching and target-line prefetching, are implemented in both the analysis and instrumentation routine, while the wrong-path prefetching scheme was implemented using only instrumentation.

Pin can be downloaded [here](https://software.intel.com/en-us/articles/pin-a-binary-instrumentation-tool-downloads).
The original icache.cpp can be downloaded [here](https://github.com/amandazhuyilan/Huntington-Ave-icache/blob/master/icache.cpp) or under pin/source/tools/Memory.

<a name="next_line_prefetching"></a>
#### Next Line Prefetching 
Next line prefetching gets the next address of the current if it is not in the cache, and the distance between the current address and the following one falls within a fetchahead distance. The 'LoadSingle' function below is modified to fetch the next address when a miss occurs and the fetchahead distance is less than twice the address size.

```c++
VOID LoadSingle(ADDRINT addr, ADDRINT next_addr, UINT32 size, UINT32 instId)
{
    const BOOL Hit = il1->AccessSingleLine(next_addr, CACHE_BASE::ACCESS_TYPE_LOAD);
    if (Hit == 0)
        {
                if ((next_addr - addr) < 2 * size){
                    const BOOL MyHit = il1->AccessSingleLine(addr, CACHE_BASE::ACCESS_TYPE_LOAD);
                    const COUNTER counter = MyHit ? COUNTER_HIT : COUNTER_MISS;
                    profile[instId][counter]++;
                }
         }
}
```

<a name="target_line_prefetching"></a>
#### Target Line Prefetching
Target-line prefetching has the ability to prefetch non-sequential cache lines. This method utilizes a target table (size set as 100 in the code snippet), with two entries on each row, address and successor address, to look up the previously entered cache lines and the next cache lines. The current address cache line is looked up in the target table, if it exists, the successor would be prefetched into the cache, if not, it would update the target table and would be available if the address is accessed once again. 

```c++
VOID LoadSingle(ADDRINT next_addr, ADDRINT addr, UINT32 instId)
{
      	unsigned long int Target_table[100][2] = {0};
        BOOL target;
        int I, n = 0;
        for (int i = 0; i < 100; i++)
        {
            target = (addr != Target_table[i][0]);
            if (target == 0)
                I = i;
        }
            if (target)
            {
                Target_table[n][0] = addr;
                Target_table[n][1] = next_addr;
                const BOOL Hit1 = il1->AccessSingleLine(addr, CACHE_BASE::ACCESS_TYPE_LOAD);
                const COUNTER counter = Hit1 ? COUNTER_HIT : COUNTER_MISS;
                profile[instId][counter]++;
                n++;
            }

            else
            {
                const BOOL Hit2 = il1->AccessSingleLine(Target_table[I][1], CACHE_BASE::ACCESS_TYPE_LOAD);
                const COUNTER counter = Hit2 ? COUNTER_HIT : COUNTER_MISS;
                profile[instId][counter]++;
            }
}
```

<a name="wrong_path_prefetching"></a>
#### Wrong Path Prefetching
Wrong-path prefetching is a combination of next-line and target-line prefetching, while it focuses mainly on the wrong paths. The successor line would be prefetched when the instructions are within the fetchahead distance, as mentioned in the next-line prefetching scheme. However, the target line address are not accessed through the pre-maintained target table. Once a brunch instruction is recognized, the target address line is prefetched immediately. If the instruction line is not a branch, the next “natural” address would be prefetched using the next- line prefetching method.


The advantage of using the wrong-path prefetching is that compared to target-line prefetching it would require less hardware resources, while it would be more accurate and efficient compared to the next-line prefetching. All branches are fetched without considering the predicted direction.

```c++
VOID Instruction(INS ins, void * v)
{
    // map sparse INS addresses to dense IDs
    const ADDRINT iaddr = INS_Address(ins);
    const UINT32 instId = profile.Map(iaddr);

    const UINT32 size   = INS_Size(ins);
    const BOOL   single = (size <= 4);

    if (KnobTrackInsts) {
        if (single)
        {
                if(INS_IsBranch(ins) && INS_HasFallThrough(ins)){ //if is taken branch
                    BOOL my_Hit1 = il1->AccessSingleLine(INS_DirectBranchOrCallTargetAddress(ins), CACHE_BASE::ACCESS_TYPE_LOAD);
                    const COUNTER counter = my_Hit1 ? COUNTER_HIT : COUNTER_MISS;
                    profile[instId][counter]++;
                }
                else{
                    BOOL my_Hit2 = il1->AccessSingleLine(INS_NextAddress(ins), CACHE_BASE::ACCESS_TYPE_LOAD);
                    const COUNTER counter = my_Hit2 ? COUNTER_HIT : COUNTER_MISS;
                    profile[instId][counter]++;
                }
        }
        else {
            INS_InsertPredicatedCall(ins, IPOINT_BEFORE, (AFUNPTR) LoadMulti,
                                     IARG_UINT32, iaddr,
                                     IARG_UINT32, size,
                                     IARG_UINT32, instId,
                                     IARG_END);
        }
    }
    else {
        if (single) {
            INS_InsertPredicatedCall(ins, IPOINT_BEFORE, (AFUNPTR) LoadSingleFast,
                                     IARG_UINT32, iaddr,
                                     IARG_END);
        }
        else {
            INS_InsertPredicatedCall(ins, IPOINT_BEFORE, (AFUNPTR) LoadMultiFast,
                                     IARG_UINT32, iaddr,
                                     IARG_UINT32, size,
                                     IARG_END);
        }
    }
}
```
### Direct Mapping, 2 and 4-way Set Associative
3 types of cache structures are comparied for analysing the efficiency of the prefeching methods.

- Direct Mapping: A cache block can only go in one spot in the cache. Cache blocks are straightforward and very easy to
find, but it‛s not very flexible and robust structure.

- 2-way Set Associative: The cache sets are structured in way where the sets that can fit
two blocks each. The index is used to find the set, and the tag helps find the block within the set.

- 4-way Set Associative: With fewer sets, each set here fits four blocks, fewer index bits are needed.

### Conclusions

It is no surprise that direct-mapping hit rates are exponentially higher than when using two and four way set associative. The average miss rates when using direct-mapping for both prefetching algorithms are more than 20% (Next-line: 21.61%, target-line 24.23%). This is reasonable as each set has more blocks, there’s a less chance of a conflict between two addresses.

The trends for the FindPi benchmark was similar of that of the Barnes benchmark: very high miss rates for direct-mapping, and not much difference between the next-line, target-line and wrong-path prefetching method. The reason for the limited difference that was obtained is that the wrong-path method works better only when there exists a large disparity between the CPU cycle time and the memory speed. The reason for this would be that the memory might not be fast enough for the prefetching to start, which is designed to prefetch target addresses that would be executed almost immediately.

One more thing to be considered: only instructions that are single (size ≤ 4) are implemented with the prefetching schemes. In other words, in other multi-instructions operations, there might be a considerable room for improvement that could be done to increase hit rates. 
