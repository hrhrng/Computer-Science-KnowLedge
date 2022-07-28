```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
    // 记录轮询信息，以method为单位的map
    ConcurrentMap<String, WeightedRoundRobin> map = methodWeightMap.computeIfAbsent(key, k -> new 																				ConcurrentHashMap<>());
    int totalWeight = 0;
    // 未知
    long maxCurrent = Long.MIN_VALUE;
    long now = System.currentTimeMillis();
    Invoker<T> selectedInvoker = null;
    WeightedRoundRobin selectedWRR = null;
    for (Invoker<T> invoker : invokers) {
        // 如果当前 Invoker 没有相应的 WeightRoundRobin ,就创建
        String identifyString = invoker.getUrl().toIdentityString();
        int weight = getWeight(invoker, invocation);
       
        WeightedRoundRobin weightedRoundRobin = map.computeIfAbsent(identifyString, k -> {
            WeightedRoundRobin wrr = new WeightedRoundRobin();
            wrr.setWeight(weight);
            return wrr;
        });
		
        // 更新 weight
        if (weight != weightedRoundRobin.getWeight()) {
            //weight changed
            weightedRoundRobin.setWeight(weight);
        }
        
        // 更新current
        long cur = weightedRoundRobin.increaseCurrent();
        weightedRoundRobin.setLastUpdate(now);
        // 找到current最大的服务器
        if (cur > maxCurrent) {
            maxCurrent = cur;
            selectedInvoker = invoker;
            selectedWRR = weightedRoundRobin;
        }
        totalWeight += weight;
    }
    // 过滤掉长时间未被更新的节点
    if (invokers.size() != map.size()) {
        map.entrySet().removeIf(item -> now - item.getValue().getLastUpdate() > RECYCLE_PERIOD);
    }
    if (selectedInvoker != null) {
        // 被选中，则减去权重综合
        selectedWRR.sel(totalWeight);
        return selectedInvoker;
    }
    // should not happen here
    return invokers.get(0);
}
```