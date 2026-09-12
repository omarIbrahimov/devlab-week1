# Customer Segmentation with RFM and Clustering 

made by Omar Ibrahimov


# Feature Preparation

1. Cleaning:
	- Delete rows that have no CustomerID because they will be useless when we will create the RFM
	- Delete rows where Invoice starts with "C"(Cancelled) because they are not purchases
	- Delete rows where Quantity is <=0 because they are returns
	- Delete rows where UnitPrice is <=0 because the transaction was either error or costed 0

In the end, we get approximately 400.000 rows of data


2. Create the RFM:
	Recency - put reference_date as the date of the very last purchase in the whole dataset + 1(to avoid 0 recency). Then just subtract the last purchase date from reference date for each customer
	Frequency - the number of unique transactions (nunique)
	Monetary - the sum of all transactions

3. Skew and scaling:
	- For skewness, I used logarithmic formula (log of x+1) we have mostly over zero values and initial skewness was too right sided.
	- For scaling, I used StandardScaler because our R,F,M were in the normal range (between -0.5 and 0.5) and we did not have any specifications for mean and variance.
	- Statistics before and after:
	--- Initial Skewness ---
Recency       1.246048
Frequency    12.067031
Monetary     19.324953

		--- Skewness After log1p ---
		Recency     -0.379169
		Frequency    1.208652
		Monetary     0.393553

		--- Scaled Features Summary ---
		                  mean  std    min    50%    max
		Recency_Scaled    -0.0  1.0 -2.341  0.090  1.564
		Frequency_Scaled  -0.0  1.0 -0.955 -0.362  5.859
		Monetary_Scaled    0.0  1.0 -4.005 -0.062  4.732

4. Clustering methods:
	- K-Means: Initially I wanted to try YellowBrick but importing was a great issue, therefore, I switched to SciKit Learn. There was a big break in the Silhouette Analysis at k = 3 and k = 4. I chose 4 as my number of clusters.
	- Hierarchial: This clustering method also gave best results for k=4. However, the number of clusters could be increased up to 6 without any noticable issue.
	- DBSCAN: I chose my minimum number of samples as 6 because the formula is 2 multiplied by the number of features which is 3 (R, F and M). Because the mean and variance were 0 and 1 respectively, I chose my esp as 0.5.
	- Choice: The best result for Silhouette Analysis was for K-means. The best result for Davies-Bouldin was DBSCAN. Therefore, I applied additional Calinski-Harabasz test and the winner was K-means with k=4.

5. Business meaning:
	- 4 categories of buyers were determined
		a. VIP
		b. Loyal
		c. Average
		d. One-time buyers
	- The largest group is one-time buyers, but the most money came from VIPs. Fun fact: One-time buyers' revenue share is higher than average buyers' even though their recency is 10 times bigger.

6. Bonus:
	- t-SNE scan was made for more detailed visualization
