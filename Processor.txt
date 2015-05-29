/*
*	Programmer:     Jonathan Reyes
*	Date:           April 9, 2015
*	Description:    Optimize a simulated scheduler that assigns a set of random, virtual processes to five virtual processors with varying speeds and limits.
*	Compile:        gcc Scheduler.c
*	Execute:        ./a.out
*/
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
#include <math.h>
#include <time.h>
#define CSE_CLOCK_SPEED 2200 // 2.2 GHZ
#define PE_CLOCK_SPEED  4000 // 4 GHZ
#define PD_CLOCK_SPEED  3000 // 3 GHZ
#define PC_CLOCK_SPEED  3000 // 3 GHZ
#define PB_CLOCK_SPEED  2000 // 2 GHZ
#define PA_CLOCK_SPEED  2000 // 2 GHZ
#define NUM_PROCESSES 50     // Number of Processes
#define NUM_PROCESSORS 5     // Number of Processors
#define MEM 7999	     // Memory
#define BURST 49999990	     // Cycles
#define MEM_OFFSET .25	     // Offset of Range
#define BURST_OFFSET exp (1) // Offset of Range

// Struct to describe an individual process
typedef struct {
    float memory;
    float burst;
    int PPID;
} Process;

// Struct to describe a processor
typedef struct {
    int frequency;
    int memory;
    float burstTotal;
} Processor;

Processor processor[NUM_PROCESSORS];

// Find the shortest burst time
void ShortestRemainingTime(Process *P);
// Print process information
void print(Process *P);
// Print
void printMap(int P[][NUM_PROCESSES/NUM_PROCESSORS+4]);
// Find the smallest process (memory)
void findSmallest(Process *P);
// Find the largest process (memory)
void findLargest(float *P);

// Mergesort declarations
void merge(Process [], int, int, int);
void mergesort(Process [], int low, int high);

int main()
{
    int i, j;

    srand(time(NULL));
    Process P[NUM_PROCESSES];
    // Initialize array of processors
    for (i = 0; i < NUM_PROCESSORS; i++){
        processor[i].frequency = CSE_CLOCK_SPEED;
        processor[i].memory = MEM;
    }
    // Specific clock speeds
    processor[0].frequency = PA_CLOCK_SPEED;
    processor[1].frequency = PB_CLOCK_SPEED;
    processor[2].frequency = PC_CLOCK_SPEED;
    processor[3].frequency = PD_CLOCK_SPEED;
    processor[4].frequency = PE_CLOCK_SPEED;

    // Initialize array of random processes
    for(i = 0; i < NUM_PROCESSES; i++){
        P[i].memory = (rand()%MEM);
        if (P[i].memory < MEM_OFFSET)
            P[i].memory += MEM_OFFSET;
        P[i].burst  = (rand()%BURST) + BURST_OFFSET;
        P[i].PPID   = i;
    }

    // Sort array of processes
    mergesort(P, 0, NUM_PROCESSES-1);

    // Find shortest time
    ShortestRemainingTime(P);
    // Print
    print(P);
    // Find smallest time
    findSmallest(P);

    return 0;
}

void ShortestRemainingTime(Process *P)
{
    // Table of Processors with their queue of processes
    int map[NUM_PROCESSORS][NUM_PROCESSES/NUM_PROCESSORS+4] = {0};
    int i, j;

    // Processors A and B are scheduled with processes
    for (i = 0; i < NUM_PROCESSES/NUM_PROCESSORS+1; i++) {
        map[0][i] = P[i * (NUM_PROCESSES/NUM_PROCESSORS-2)].burst;
        map[1][i] = P[i * (NUM_PROCESSES/NUM_PROCESSORS-2) + 1].burst;
    }

    // Processors C-E are scheduled with processes
    for (i = 0, j = 0; i < NUM_PROCESSES; i += NUM_PROCESSES/NUM_PROCESSORS-2, j += 2) {
        map[2][j] = P[i+2].burst;
        map[3][j] = P[i+3].burst;
        map[4][j] = P[i+4].burst;
        map[4][j+1] = P[i+5].burst;
        map[3][j+1] = P[i+6].burst;
        map[2][j+1] = P[i+7].burst;
    }

    // Print the assigned processes
    printMap(map);

    // For each Processor
    for (i = 0; i < NUM_PROCESSORS; i++)
        // Processors A and B
        if (processor[i].frequency == PA_CLOCK_SPEED) {
        // Add up all of the process burst times for Processors A and B
        for(j = 0; j < (NUM_PROCESSES/NUM_PROCESSORS)-5; j++){
            processor[i].burstTotal += map[i][j];
        }
    } else {
        // Add up all of the process burst times for Processors C - E
        for(j = 0; j < (NUM_PROCESSES/NUM_PROCESSORS); j++){
            processor[i].burstTotal += map[i][j];
        }
    }

    float total[NUM_PROCESSORS] = {0};
    // Find the average time for each processor to execute all processes
    for (i = 0; i < NUM_PROCESSORS; i++){
        total[i] = processor[i].burstTotal/processor[i].frequency;
        printf("Processor %d: Time: %f\n", i, processor[i].burstTotal/processor[i].frequency);
    }
    // Find the longest turnaround time among the five processors
    findLargest(total);
}

// Find the smallest turnaround time among the five processors
void findSmallest(Process *P)
{
    int i;
    int   id = P[0].PPID;
    float small = P[0].burst;
    for(i = 0; i < NUM_PROCESSES; i++){
        if(P[i].burst < small){
            small = P[i].burst;
            id = P[i].PPID;
        }
    }
}

// Find the longest turnaround time among the five processors
void findLargest(float *P)
{
    int i;
    float big = P[0];
    for(i = 0; i < NUM_PROCESSORS; i++){
        if(P[i] > big){
            big = P[i];
        }
    }

    printf("Longest Turnaround time: %f\n", big);
}

void print(Process *P)
{
    int i;
    for(i = 0; i < NUM_PROCESSES; i++)
        printf("PID: %d\t Memory: %.2f   \t Burst: %.0f \n",P[i].PPID, P[i].memory, P[i].burst);
}

void printMap(int P[][NUM_PROCESSES/NUM_PROCESSORS+4])
{
    int i, j;
    for (i = 0; i < NUM_PROCESSORS; i++){
        for(j = 0; j < (NUM_PROCESSES/NUM_PROCESSORS+2); j++)
            printf("\nProcessor %d: Burst: %d", i, P[i][j]);
        printf("\n");
    }
}

// Sort the processes with respect to burst times
void mergesort(Process a[], int low, int high)
{
    int mid;
    if(low<high) {
        mid=(low+high)/2;
        mergesort(a,low,mid);
        mergesort(a,mid+1,high);
        merge(a,low,high,mid);
    }
}

void merge(Process a[], int low, int high, int mid)
{
    int i, j, k;
    Process c[NUM_PROCESSES];
    i=low;
    j=mid+1;
    k=low;

    while((i<=mid)&&(j<=high)) {
        if(a[i].burst<a[j].burst) {
            c[k]=a[i];
            k++;
            i++;
        } else {
            c[k]=a[j];
            k++;
            j++;
        }
    }

    while(i<=mid) {
        c[k]=a[i];
        k++;
        i++;
    }

    while(j<=high) {
        c[k]=a[j];
        k++;
        j++;
    }

    for(i=low;i<k;i++) {
        a[i]=c[i];
    }
}
