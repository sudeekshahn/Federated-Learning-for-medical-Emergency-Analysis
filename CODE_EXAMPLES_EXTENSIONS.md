# Healthcare Scheduling System - Code Examples & Extensions

## Table of Contents
1. Code Examples
2. Extension Templates
3. Advanced Implementations
4. Testing Utilities

---

## Part 1: Code Examples

### Example 1: Creating Custom Medical Task

```java
// Define a custom heart disease diagnostic task
PatientTask createHeartDiseaseTask(int taskId, int brokerId) {
    // Task specifications
    String taskType = "CARDIAC_IMAGING";
    long taskLength = 750000;  // Complex imaging processing
    int pesNumber = 2;         // Multi-core processing
    long fileSize = 2000;      // Large medical image files
    long outputSize = 500;     // Report generation
    int emergencyLevel = 2;    // High priority (heart disease)
    String patientId = "PAT_CARDIAC_001";
    String department = "Cardiology";
    
    // Create task
    UtilizationModel utilizationModel = new UtilizationModelFull();
    PatientTask task = new PatientTask(taskId, taskLength, pesNumber,
        fileSize, outputSize, utilizationModel,
        taskType, emergencyLevel, patientId, department);
    
    // Configure for broker
    task.setUserId(brokerId);
    task.setGuestId(1);  // Cardiology VM
    task.setEstimatedCost(5.0);  // $5 per hour
    
    return task;
}
```

### Example 2: Custom VM Creation with Enhanced Specifications

```java
// Create specialized VM for emergency department
Vm createEmergencyVM(int vmId, int brokerId) {
    int mips = 3000;           // Highest performance
    int cores = 8;             // Maximum cores
    int ram = 4096;            // 4GB RAM
    long bw = 10000;           // High bandwidth
    long storage = 100000;     // Fast storage
    String vmm = "KVM";
    
    // Use time-shared scheduler for priority handling
    CloudletScheduler scheduler = new CloudletSchedulerTimeShared();
    
    Vm emergencyVM = new Vm(vmId, brokerId, mips, cores, ram, bw,
        storage, vmm, scheduler);
    
    return emergencyVM;
}
```

### Example 3: Implementing Custom Scheduling Logic

```java
// Create custom broker with priority handling
class HealthcareBroker extends DatacenterBroker {
    private PriorityQueue<Cloudlet> emergencyQueue;
    private Queue<Cloudlet> normalQueue;
    
    public HealthcareBroker(String name) throws Exception {
        super(name);
        emergencyQueue = new PriorityQueue<>(
            Comparator.comparingInt(c -> 
                -((PatientTask)c).getPriorityLevel()
            )
        );
        normalQueue = new LinkedList<>();
    }
    
    // Override cloudlet submission to implement custom queuing
    public void submitCloudletList(List<Cloudlet> cloudletList) {
        for (Cloudlet cloudlet : cloudletList) {
            PatientTask task = (PatientTask) cloudlet;
            if (task.getPriorityLevel() >= 2) {
                emergencyQueue.offer(cloudlet);
            } else {
                normalQueue.offer(cloudlet);
            }
        }
        
        // Submit emergency tasks first
        List<Cloudlet> orderedList = new ArrayList<>();
        orderedList.addAll(emergencyQueue);
        orderedList.addAll(normalQueue);
        
        super.submitCloudletList(orderedList);
    }
}
```

### Example 4: Performance Monitoring

```java
// Real-time performance tracking
class PerformanceMonitor {
    private static class TaskMetrics {
        int taskId;
        long submissionTime;
        long startTime;
        long completionTime;
        int priority;
        String taskType;
        
        public long getWaitTime() {
            return startTime - submissionTime;
        }
        
        public long getExecutionTime() {
            return completionTime - startTime;
        }
        
        public long getTurnaroundTime() {
            return completionTime - submissionTime;
        }
    }
    
    private List<TaskMetrics> metrics = new ArrayList<>();
    
    public void recordTaskStart(PatientTask task) {
        TaskMetrics m = new TaskMetrics();
        m.taskId = task.getCloudletId();
        m.submissionTime = task.getExecStartTime();
        m.startTime = System.currentTimeMillis();
        m.priority = task.getPriorityLevel();
        m.taskType = task.getTaskType();
        metrics.add(m);
    }
    
    public void recordTaskCompletion(PatientTask task) {
        for (TaskMetrics m : metrics) {
            if (m.taskId == task.getCloudletId()) {
                m.completionTime = System.currentTimeMillis();
                break;
            }
        }
    }
    
    public void printReport() {
        double avgWait = metrics.stream()
            .mapToLong(TaskMetrics::getWaitTime)
            .average()
            .orElse(0.0);
        
        double avgExec = metrics.stream()
            .mapToLong(TaskMetrics::getExecutionTime)
            .average()
            .orElse(0.0);
        
        Log.println("Average Wait Time: " + avgWait + "ms");
        Log.println("Average Execution Time: " + avgExec + "ms");
        
        // Print emergency tasks separately
        metrics.stream()
            .filter(m -> m.priority >= 2)
            .forEach(m -> Log.println(
                "Emergency Task " + m.taskId + 
                ": Wait=" + m.getWaitTime() + "ms"
            ));
    }
}
```

### Example 5: Cost Calculation

```java
// Advanced cost calculation with multiple factors
class HealthcareCostCalculator {
    final double BASE_COST_PER_HOUR = 10.0;
    final double PRIORITY_MULTIPLIER_NORMAL = 1.0;
    final double PRIORITY_MULTIPLIER_HIGH = 1.2;
    final double PRIORITY_MULTIPLIER_EMERGENCY = 1.5;
    final double TIME_OF_DAY_PEAK = 1.3;      // Peak hours cost more
    final double TIME_OF_DAY_OFF_PEAK = 0.8;  // Off-peak discount
    
    public double calculateTaskCost(PatientTask task, int hour) {
        // Base execution cost
        double executionCost = (task.getActualCPUTime() / 3600.0) * BASE_COST_PER_HOUR;
        
        // Apply priority multiplier
        double priorityMultiplier = getPriorityMultiplier(task.getPriorityLevel());
        
        // Apply time-of-day multiplier
        double timeMultiplier = getTimeMultiplier(hour);
        
        // Apply resource usage multiplier
        double resourceMultiplier = 1.0 + (task.getCloudletLength() / 1000000.0) * 0.1;
        
        // Calculate final cost
        double finalCost = executionCost * priorityMultiplier * 
                          timeMultiplier * resourceMultiplier;
        
        return finalCost;
    }
    
    private double getPriorityMultiplier(int priority) {
        switch(priority) {
            case 3: return PRIORITY_MULTIPLIER_EMERGENCY;
            case 2: return PRIORITY_MULTIPLIER_HIGH;
            default: return PRIORITY_MULTIPLIER_NORMAL;
        }
    }
    
    private double getTimeMultiplier(int hour) {
        // Peak hours: 8-18
        if (hour >= 8 && hour <= 18) {
            return TIME_OF_DAY_PEAK;
        } else {
            return TIME_OF_DAY_OFF_PEAK;
        }
    }
    
    public void printCostBreakdown(PatientTask task, int hour) {
        double base = (task.getActualCPUTime() / 3600.0) * BASE_COST_PER_HOUR;
        double priority = getPriorityMultiplier(task.getPriorityLevel());
        double time = getTimeMultiplier(hour);
        double total = calculateTaskCost(task, hour);
        
        Log.println("Cost Breakdown for Task " + task.getCloudletId() + ":");
        Log.println("  Base Cost: $" + String.format("%.2f", base));
        Log.println("  Priority Multiplier: " + priority);
        Log.println("  Time-of-Day Multiplier: " + time);
        Log.println("  Final Cost: $" + String.format("%.2f", total));
    }
}
```

---

## Part 2: Extension Templates

### Template 1: Multi-Hospital Federated System

```java
// Simulate multiple hospitals sharing cloud resources
class FederatedHealthcareSystem {
    
    class Hospital {
        String name;
        DatacenterBroker broker;
        List<Datacenter> datacenters;
        List<Vm> vms;
        
        public Hospital(String name, int vmCount) {
            this.name = name;
            this.vms = new ArrayList<>();
        }
        
        public void submitTasks(List<PatientTask> tasks) {
            // Tasks stay local if resources available
            // Otherwise, route to partner hospitals
            broker.submitCloudletList(tasks);
        }
    }
    
    private List<Hospital> hospitals = new ArrayList<>();
    
    public void createFederatedSystem() {
        // Create network of hospitals
        Hospital hospital1 = new Hospital("City_Hospital", 5);
        Hospital hospital2 = new Hospital("Rural_Clinic", 3);
        Hospital hospital3 = new Hospital("Trauma_Center", 8);
        
        hospitals.add(hospital1);
        hospitals.add(hospital2);
        hospitals.add(hospital3);
    }
    
    public void routeEmergencyTask(PatientTask task) {
        // Find hospital with capacity
        Hospital best = hospitals.stream()
            .min(Comparator.comparingInt(h -> h.broker.getCloudletReceivedList().size()))
            .orElse(hospitals.get(0));
        
        best.submitTasks(Arrays.asList(task));
        Log.println("Emergency routed to: " + best.name);
    }
}
```

### Template 2: Machine Learning-Based Priority Assignment

```java
// Use historical data to predict task priority
class MLPriorityPredictor {
    
    // Simulated ML model (in production, use TensorFlow/PyTorch)
    static class PatientRiskModel {
        public int predictEmergencyLevel(String patientId, String symptoms) {
            // In production: Call actual ML model
            // This is simulated for demonstration
            
            if (symptoms.contains("chest") || symptoms.contains("difficulty breathing")) {
                return 3; // Emergency
            } else if (symptoms.contains("severe")) {
                return 2; // High
            } else if (symptoms.contains("urgent")) {
                return 1; // Normal
            }
            return 0; // Low
        }
    }
    
    private PatientRiskModel model;
    
    public PatientTask createTaskWithPredictedPriority(
            int taskId, String patientId, String symptoms, int brokerId) {
        
        // Predict priority from symptoms
        int priority = model.predictEmergencyLevel(patientId, symptoms);
        
        // Create task with predicted priority
        PatientTask task = new PatientTask(
            taskId, 500000, 1, 300, 300,
            new UtilizationModelFull(),
            "DIAGNOSTIC", priority, patientId, "Emergency"
        );
        task.setUserId(brokerId);
        
        return task;
    }
}
```

### Template 3: Dynamic Scheduling Adjustment

```java
// Adjust scheduling based on real-time metrics
class AdaptiveScheduler {
    
    class SystemState {
        double avgWaitTime;
        double systemUtilization;
        int emergencyQueueSize;
        double costPerTask;
    }
    
    private SystemState currentState;
    
    public void monitorAndAdjust() {
        currentState = collectMetrics();
        
        // If emergency queue is backing up
        if (currentState.emergencyQueueSize > 5) {
            Log.println("ALERT: Emergency queue backing up!");
            recommendActions();
        }
        
        // If costs are too high
        if (currentState.costPerTask > 100) {
            Log.println("ALERT: Costs exceeding threshold!");
            recommendCostOptimization();
        }
    }
    
    private void recommendActions() {
        Log.println("Recommended Actions:");
        Log.println("1. Increase VM cores for emergency department");
        Log.println("2. Preempt non-critical tasks");
        Log.println("3. Route to partner hospitals if available");
        Log.println("4. Alert hospital management");
    }
    
    private void recommendCostOptimization() {
        Log.println("Cost Optimization Suggestions:");
        Log.println("1. Defer non-urgent tasks to off-peak hours");
        Log.println("2. Batch similar tasks for efficiency");
        Log.println("3. Use reserved instances instead of on-demand");
    }
    
    private SystemState collectMetrics() {
        SystemState state = new SystemState();
        // Collect current metrics from simulation
        return state;
    }
}
```

### Template 4: SLA Enforcement

```java
// Enforce Service Level Agreements
class SLAEnforcer {
    
    static class ServiceLevelAgreement {
        int priorityLevel;
        int maxWaitTimeSeconds;
        int maxTurnaroundTimeSeconds;
        double minCompliancePercent;
        
        public SLAEnforcer(int priority, int wait, int turnaround, double compliance) {
            this.priorityLevel = priority;
            this.maxWaitTimeSeconds = wait;
            this.maxTurnaroundTimeSeconds = turnaround;
            this.minCompliancePercent = compliance;
        }
    }
    
    private List<ServiceLevelAgreement> slas = new ArrayList<>();
    
    public void defineSLAs() {
        // Emergency SLA: Complete within 5 minutes
        slas.add(new ServiceLevelAgreement(3, 60, 300, 95.0));
        
        // High Priority SLA: Complete within 15 minutes
        slas.add(new ServiceLevelAgreement(2, 180, 900, 90.0));
        
        // Normal SLA: Complete within 1 hour
        slas.add(new ServiceLevelAgreement(1, 600, 3600, 85.0));
        
        // Low Priority SLA: Complete within 4 hours
        slas.add(new ServiceLevelAgreement(0, 1200, 14400, 80.0));
    }
    
    public void validateSLACompliance(List<PatientTask> completedTasks) {
        for (ServiceLevelAgreement sla : slas) {
            List<PatientTask> relevantTasks = completedTasks.stream()
                .filter(t -> t.getPriorityLevel() == sla.priorityLevel)
                .collect(Collectors.toList());
            
            if (relevantTasks.isEmpty()) continue;
            
            long compliant = relevantTasks.stream()
                .filter(t -> (t.getExecStartTime() + t.getActualCPUTime()) 
                        <= sla.maxTurnaroundTimeSeconds)
                .count();
            
            double compliancePercent = (compliant * 100.0) / relevantTasks.size();
            
            String status = compliancePercent >= sla.minCompliancePercent ? "✓ PASS" : "✗ FAIL";
            Log.println("Priority " + sla.priorityLevel + " SLA: " + 
                       String.format("%.1f%%", compliancePercent) + " " + status);
        }
    }
}
```

---

## Part 3: Advanced Implementations

### Advanced Implementation 1: Hybrid Scheduling

```java
// Combine multiple scheduling strategies
class HybridScheduler {
    
    enum SchedulingPhase {
        EMERGENCY_FIRST,      // Priority 3 and 2
        AGING_FAIRNESS,       // Boost old tasks
        LOAD_BALANCING        // Distribute evenly
    }
    
    private SchedulingPhase currentPhase = SchedulingPhase.EMERGENCY_FIRST;
    private int tasksProcessed = 0;
    private final int PHASE_SWITCH_THRESHOLD = 50;
    
    public void scheduleTask(PatientTask task, List<Vm> vmList) {
        switch(currentPhase) {
            case EMERGENCY_FIRST:
                scheduleByPriority(task, vmList);
                break;
            case AGING_FAIRNESS:
                scheduleWithAging(task, vmList);
                break;
            case LOAD_BALANCING:
                scheduleByLoadBalance(task, vmList);
                break;
        }
        
        tasksProcessed++;
        if (tasksProcessed % PHASE_SWITCH_THRESHOLD == 0) {
            rotatePhase();
        }
    }
    
    private void scheduleByPriority(PatientTask task, List<Vm> vmList) {
        // Process high-priority tasks first
        if (task.getPriorityLevel() >= 2) {
            vmList.get(0).submitCloudlet(task);  // Priority VM
        } else {
            vmList.get(1).submitCloudlet(task);  // Regular VMs
        }
    }
    
    private void scheduleWithAging(PatientTask task, List<Vm> vmList) {
        // Boost priority of waiting tasks
        if (task.getExecStartTime() > 300) {  // Waited > 5 minutes
            task.setExecParam(task.getPriorityLevel() + 1, 0);
        }
        scheduleByPriority(task, vmList);
    }
    
    private void scheduleByLoadBalance(PatientTask task, List<Vm> vmList) {
        // Send to least loaded VM
        Vm leastLoaded = vmList.stream()
            .min(Comparator.comparingDouble(Vm::getNumberOfPes))
            .orElse(vmList.get(0));
        leastLoaded.submitCloudlet(task);
    }
    
    private void rotatePhase() {
        SchedulingPhase[] phases = SchedulingPhase.values();
        int nextIndex = (currentPhase.ordinal() + 1) % phases.length;
        currentPhase = phases[nextIndex];
        Log.println("Switching to: " + currentPhase);
    }
}
```

### Advanced Implementation 2: Geographic Distribution

```java
// Route tasks based on geography and SLA
class GeographicScheduler {
    
    class Hospital {
        String name;
        String location;
        double latitude, longitude;
        Datacenter datacenter;
        
        public double getDistanceTo(double lat, double lon) {
            // Haversine formula for distance
            double R = 6371; // Earth radius in km
            double dLat = Math.toRadians(lat - latitude);
            double dLon = Math.toRadians(lon - longitude);
            double a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
                      Math.cos(Math.toRadians(latitude)) * Math.cos(Math.toRadians(lat)) *
                      Math.sin(dLon / 2) * Math.sin(dLon / 2);
            return 2 * R * Math.asin(Math.sqrt(a));
        }
    }
    
    private List<Hospital> hospitals = new ArrayList<>();
    
    public void routeTask(PatientTask task, double patientLat, double patientLon) {
        Hospital chosen;
        
        if (task.getPriorityLevel() == 3) {
            // Emergency: Send to nearest hospital
            chosen = hospitals.stream()
                .min(Comparator.comparingDouble(h -> 
                    h.getDistanceTo(patientLat, patientLon)
                ))
                .orElse(hospitals.get(0));
        } else {
            // Regular: Send to hospital with capacity
            chosen = hospitals.stream()
                .min(Comparator.comparingInt(h -> 
                    h.datacenter.getCharacteristics().getNumberOfPes()
                ))
                .orElse(hospitals.get(0));
        }
        
        Log.println("Task routed to: " + chosen.name + 
                   " (Distance: " + String.format("%.1f", 
                   chosen.getDistanceTo(patientLat, patientLon)) + " km)");
    }
}
```

---

## Part 4: Testing Utilities

### Testing Utility 1: Simulation Validator

```java
// Verify simulation correctness
class SimulationValidator {
    
    public static void validateResults(List<PatientTask> tasks) {
        boolean valid = true;
        
        for (PatientTask task : tasks) {
            // Check 1: Turnaround = Wait + Execution
            double calculated = task.getExecStartTime() + task.getActualCPUTime();
            double expected = task.getFinishTime() - task.getSubmissionTime();
            if (Math.abs(calculated - expected) > 1.0) {
                Log.println("ERROR: Turnaround mismatch for task " + task.getCloudletId());
                valid = false;
            }
            
            // Check 2: Execution time not negative
            if (task.getActualCPUTime() < 0) {
                Log.println("ERROR: Negative execution time for task " + task.getCloudletId());
                valid = false;
            }
            
            // Check 3: Status is SUCCESS
            if (task.getStatus() != Cloudlet.CloudletStatus.SUCCESS) {
                Log.println("ERROR: Task " + task.getCloudletId() + " not successful");
                valid = false;
            }
        }
        
        if (valid) {
            Log.println("✓ All validation checks passed!");
        } else {
            Log.println("✗ Validation failed - review results");
        }
    }
}
```

### Testing Utility 2: Benchmark Runner

```java
// Run multiple simulations and compare results
class BenchmarkRunner {
    
    public static void runBenchmark() {
        Log.println("Running Healthcare Scheduler Benchmark...\n");
        
        String[] algorithms = {"FCFS", "RoundRobin", "PriorityBased"};
        int[] taskCounts = {10, 50, 100, 200};
        
        for (String algo : algorithms) {
            Log.println("Algorithm: " + algo);
            for (int taskCount : taskCounts) {
                long startTime = System.currentTimeMillis();
                // Run simulation
                double result = runSimulation(algo, taskCount);
                long elapsed = System.currentTimeMillis() - startTime;
                
                Log.println("  Tasks: " + taskCount + 
                           " | Result: " + String.format("%.2f", result) +
                           " | Time: " + elapsed + "ms");
            }
            Log.println();
        }
    }
    
    private static double runSimulation(String algorithm, int taskCount) {
        // Execute simulation and return metric
        return Math.random() * 1000;  // Placeholder
    }
}
```

---

## Quick Reference

### Common Customizations

```java
// Increase emergency priority handling
if (task.emergencyLevel == 3) {
    task.setVirtualMachineMapping(emergencyVM);
}

// Add custom metrics
metrics.put("avg_emergency_response", calculateEmergencyMetric());
metrics.put("sla_compliance", calculateSLACompliance());

// Extend task properties
task.departmentName = "Radiology";
task.patientAge = 45;
task.medicalCondition = "Suspected Fracture";

// Modify scheduling order
tasks.sort((t1, t2) -> {
    PatientTask p1 = (PatientTask) t1;
    PatientTask p2 = (PatientTask) t2;
    // Primary: Priority (descending)
    int cmp = Integer.compare(p2.getPriorityLevel(), p1.getPriorityLevel());
    // Secondary: Waiting time (ascending) for aging
    if (cmp == 0) {
        cmp = Long.compare(p1.getExecStartTime(), p2.getExecStartTime());
    }
    return cmp;
});
```

