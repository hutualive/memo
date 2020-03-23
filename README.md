# freertos kernel interrupt priority on arm cortex-m3/4/7
1. priority preempt bit is vendor dependent: TI - 3 bit, NXP - 5 bit, ST - 4 bit  
2. configMAX_SYSCALL_INTERRUPT_PRIORITY: boundry of maskable interrupt by RTOS critical sections  
3. assign all priority bits as preempt bits: NVIC_PriorityGroupConfig(NVIC_PriorityGroup_4) before RTOS started   
4. configMAX_SYSCALL_INTERRUPT_PRIORITY and configKERNEL_INTERRUPT_PRIORITY(255 - 1111 1111 - lowest priority) already shifted to the most significant bits of the priority byte  
5. ISR use RTOS APT must have lower logical priority(higher numeric value) than configMAX_SYSCALL_INTERRUPT_PRIORITY  
6. RTOS kernel create a critical section by writing the configMAX_SYSCALL_INTERRUPT_PRIORITY value into the ARM Cortex-M BASEPRI register(so it value must not be set to 0)  
