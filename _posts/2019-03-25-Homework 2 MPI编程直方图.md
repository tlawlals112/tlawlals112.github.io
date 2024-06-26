---
redirect_from: /_posts/2019-03-25-Homework-2-MPI%E7%BC%96%E7%A8%8B%E7%9B%B4%E6%96%B9%E5%9B%BE/
title: Homework 2 MPI编程直方图
tags: 课程实验与项目
---
[题目链接](https://easyhpc.org/problems/program/355/)

第一次做不是ACM的题解了。

使用MPI实现2.7.1节讨论的直方图程序，进程0读取输入的数据，并将它们分配到其余进程，最后进程0打印该直方图。
请根据提供的代码，将代码补充完整！ 需要补充的部分见注释 PLEASE ADD THE RIGHT CODES 部分。可直接点击下方作业题目跳转至在线编程平台，点击参考代码，补充完整后提交。（或下载附件 prog3.1_mpi_histo(blank).zip）

在参考代码的基础上加了155行和168~175行就解决问题了。这里找对应下标的时候使用了二分，其实是可以直接算出来的。
```c
/*
 * File:     prog3.1_mpi_histo.c
 *
 * Purpose:  Use MPI to implement a program that creates a histogram
 *
 * Compile:  mpicc -g -Wall -o prog3.1_mpi_histo prog3.1_mpi_histo.c
 * Run:      mpiexec -n <comm_sz> ./prog3.1_mpi_histo
 *
 * Input:    Number of bins
 *           Minimum measurement
 *           Maximum measurement
 *           Number of data items
 *
 * Output:   Histogram of the data.  If DEBUG flag is set, the data
 *           generated.
 *
 * Notes:
 * 1.  The number of data must be evenly divisible by the number of processes
 * 2.  The program generates random floats x in the range
 *
 *        min measurement <= x < max measurement
 *
 * 3.  If i >= 1, the ith bin contains measurements x in the
 *     range
 *
 *        bin_maxes[i-1] <= x < bin_maxes[i]
 *
 *     Bin 0 will contain measurements x in the range
 *
 *        min measurement <= x < bin_maxes[0]
 *
 *
 * IPP:     Programming Assignment 3.1
 */

#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>

void Get_input(int *bin_count_p, float *min_meas_p, float *max_meas_p,
			   int *data_count_p, int *local_data_count_p, int my_rank,
			   int comm_sz, MPI_Comm comm);
void Check_for_error(int local_ok, char fname[], char message[],
					 MPI_Comm comm);
void Gen_data(float local_data[], int local_data_count, int data_count,
			  float min_meas, float max_meas, int my_rank, MPI_Comm comm);
void Set_bins(float bin_maxes[], int loc_bin_cts[], float min_meas,
			  float max_meas, int bin_count, int my_rank, MPI_Comm comm);
void Find_bins(int bin_counts[], float local_data[], int loc_bin_cts[],
			   int local_data_count, float bin_maxes[],
			   int bin_count, float min_meas, MPI_Comm comm);
int Which_bin(float data, float bin_maxes[], int bin_count,
			  float min_meas);
void Print_histo(float bin_maxes[], int bin_counts[], int bin_count,
				 float min_meas);

/*---------------------------------------------------------------------*/
int main(void)
{
	int bin_count;

	float min_meas, max_meas;
	float *bin_maxes;
	int *bin_counts;
	int *loc_bin_cts;

	int data_count;
	int local_data_count;

	float *data;
	float *local_data;

	int my_rank, comm_sz;
	MPI_Comm comm;

	MPI_Init(NULL, NULL);
	comm = MPI_COMM_WORLD;
	MPI_Comm_size(comm, &comm_sz);
	MPI_Comm_rank(comm, &my_rank);

	// get user inputs for bin_count, max_meas, min_meas, and data_count
	Get_input(&bin_count, &min_meas, &max_meas, &data_count,
			  &local_data_count, my_rank, comm_sz, comm);

	// allocate arrays
	bin_maxes = malloc(bin_count * sizeof(float));
	bin_counts = malloc(bin_count * sizeof(int));
	loc_bin_cts = malloc(bin_count * sizeof(int));
	data = malloc(data_count * sizeof(float));
	local_data = malloc(local_data_count * sizeof(float));

	Set_bins(bin_maxes, loc_bin_cts, min_meas, max_meas, bin_count,
			 my_rank, comm);
	Gen_data(local_data, local_data_count, data_count, min_meas,
			 max_meas, my_rank, comm);
	Find_bins(bin_counts, local_data, loc_bin_cts, local_data_count,
			  bin_maxes, bin_count, min_meas, comm);

	if (my_rank == 0)
		Print_histo(bin_maxes, bin_counts, bin_count, min_meas);

	free(bin_maxes);
	free(bin_counts);
	free(loc_bin_cts);
	free(data);
	free(local_data);
	MPI_Finalize();
	return 0;

} /* main */

/*---------------------------------------------------------------------*/
void Print_histo(
	float bin_maxes[] /* in */,
	int bin_counts[] /* in */,
	int bin_count /* in */,
	float min_meas /* in */)
{
	int i, j;
	float bin_max, bin_min;

	for (i = 0; i < bin_count; i++)
	{
		bin_max = bin_maxes[i];
		bin_min = (i == 0) ? min_meas : bin_maxes[i - 1];
		printf("%.3f-%.3f:\t", bin_min, bin_max);
		for (j = 0; j < bin_counts[i]; j++)
			printf("X");
		printf("\n");
	}
} /* Print_histo */

/*---------------------------------------------------------------------*/
void Find_bins(
	int bin_counts[] /* out */,
	float local_data[] /* in  */,
	int loc_bin_cts[] /* out */,
	int local_data_count /* in  */,
	float bin_maxes[] /* in  */,
	int bin_count /* in  */,
	float min_meas /* in  */,
	MPI_Comm comm)
{

	int i, bin;
	for (i = 0; i < local_data_count; i++)
	{
		bin = Which_bin(local_data[i], bin_maxes, bin_count, min_meas);
		loc_bin_cts[bin]++;
	}

	/*******************************************************************/

	//PLEASE ADD THE RIGHT CODES
	MPI_Reduce(loc_bin_cts, bin_counts, bin_count, MPI_INT, MPI_SUM, 0, comm);

	/******************************************************************/
} /* Find_bins */

/*---------------------------------------------------------------------*/
int Which_bin(float data, float bin_maxes[], int bin_count,
			  float min_meas)
{

	/*******************************************************************/

	//PLEASE ADD THE RIGHT CODES
	int b = 0, e = bin_count;
	while (e - b > 1)
	{
		int m = b + (e - b) / 2;
		data < bin_maxes[m] ? (b = m) : (e = m);
	}
	if (b)
		return b;
	/******************************************************************/

	printf("Uh oh . . .\n");
	return 0;
} /* Which_bin */

/*---------------------------------------------------------------------*/
void Set_bins(
	float bin_maxes[] /* out */,
	int loc_bin_cts[] /* out */,
	float min_meas /* in  */,
	float max_meas /* in  */,
	int bin_count /* in  */,
	int my_rank /* in  */,
	MPI_Comm comm /* in  */)
{

	if (my_rank == 0)
	{
		int i;
		float bin_width;
		bin_width = (max_meas - min_meas) / bin_count;

		for (i = 0; i < bin_count; i++)
		{
			loc_bin_cts[i] = 0;
			bin_maxes[i] = min_meas + (i + 1) * bin_width;
		}
	}

	// set bin_maxes for each proc
	MPI_Bcast(bin_maxes, bin_count, MPI_FLOAT, 0, comm);

	// reset loc_bin_cts of each proc
	MPI_Bcast(loc_bin_cts, bin_count, MPI_INT, 0, comm);
} /* Set_bins */

/*---------------------------------------------------------------------*/
void Gen_data(
	float local_data[] /* out */,
	int local_data_count /* in  */,
	int data_count /* in  */,
	float min_meas /* in  */,
	float max_meas /* in  */,
	int my_rank /* in  */,
	MPI_Comm comm /* in  */)
{
	int i;
	float *data = NULL;

	if (my_rank == 0)
	{
		data = malloc(data_count * sizeof(float));
		srandom(1);
		for (i = 0; i < data_count; i++)
			data[i] =
				min_meas + (max_meas - min_meas) * random() / ((double)RAND_MAX);
#ifdef DEBUG
		printf("Generated data:\n   ");
		for (i = 0; i < data_count; i++)
			printf("%.3f ", data[i]);
		printf("\n\n");
#endif
		MPI_Scatter(data, local_data_count, MPI_FLOAT, local_data,
					local_data_count, MPI_FLOAT, 0, comm);
		free(data);
	}
	else
	{
		MPI_Scatter(data, local_data_count, MPI_FLOAT, local_data,
					local_data_count, MPI_FLOAT, 0, comm);
	}

} /* Gen_data */

/*---------------------------------------------------------------------*/
void Get_input(int *bin_count_p, float *min_meas_p, float *max_meas_p,
			   int *data_count_p, int *local_data_count_p, int my_rank,
			   int comm_sz, MPI_Comm comm)
{

	int local_ok = 1;

	if (my_rank == 0)
	{
		printf("Enter the number of bins\n");
		scanf("%d", bin_count_p);
		printf("Enter the minimum measurement\n");
		scanf("%f", min_meas_p);
		printf("Enter the maximum measurement\n");
		scanf("%f", max_meas_p);
		printf("Enter the number of data\n");
		scanf("%d", data_count_p);
	}

	MPI_Bcast(bin_count_p, 1, MPI_INT, 0, comm);
	MPI_Bcast(min_meas_p, 1, MPI_INT, 0, comm);
	MPI_Bcast(max_meas_p, 1, MPI_INT, 0, comm);
	MPI_Bcast(data_count_p, 1, MPI_INT, 0, comm);

	if (*data_count_p % comm_sz != 0)
		local_ok = 0;

	Check_for_error(local_ok, "Get_input",
					"data_count must be evenly divisible by comm_sz",
					comm);
	*local_data_count_p = *data_count_p / comm_sz;

} /* Get_input */

/*---------------------------------------------------------------------*/
void Check_for_error(
	int local_ok /* in */,
	char fname[] /* in */,
	char message[] /* in */,
	MPI_Comm comm /* in */)
{
	int ok;

	MPI_Allreduce(&local_ok, &ok, 1, MPI_INT, MPI_MIN, comm);
	if (ok == 0)
	{
		int my_rank;
		MPI_Comm_rank(comm, &my_rank);
		if (my_rank == 0)
		{
			fprintf(stderr, "Proc %d > In %s, %s\n", my_rank, fname,
					message);
			fflush(stderr);
		}
		MPI_Finalize();
		exit(-1);
	}
} /* Check_for_error */
```
