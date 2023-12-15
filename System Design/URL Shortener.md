# Functional

- Create a short URL against an original long URL
- Redirection of short URL to the original long URL
- Expiry

# Non-functional

- Highly available: Service should be up and running all the time
- Low latency: URL redirection should be fast and should not degrade at any point of time (Even during peak loads)
- URL should not be predictable

# **Back of envelope calculation**

## Traffic Estimation

Assuming _200:1 read/write_ ratio

Number of unique shortened links generated per month = 100 million

Number of unique shortened links generated per seconds = 100 million /(30 days * 24 hours * 3600 seconds ) ~ **40 URLs/second**

With 200:1 read/write ratio, number of redirections = 40 URLs/s * 200 = **8000 URLs/s**

## **Storage Estimation** 

Assuming lifetime of service to be 100 years and with 100 million shortened links creation per month, total number of data points/objects in system will be = 100 million/month * 100 (years) * 12 (months) = 120 billion

Assuming size of each data object (_Short url, long url, created date_ etc.) to be 500 bytes long, then total require storage = 120 billion * 500 bytes =**60TB**

## **Memory Estimation**

Following Pareto Principle, better known as the 80:20 rule for caching. (80% requests are for 20% data)

Since we get 8000 read/redirection requests per second, we will be getting 700 million requests per day:

8000/s * 86400 seconds =**~700 million**

To cache 20% of these requests, we will need ~70GB of memory.

0.2 * 700 million * 500 bytes = **~70GB**


# High Level Design


![[url-shortener.jpg]]


1. Multiple instances of WebServers to prevent single point of failure (SPOF)
2. Added a load balancer in front of WebServers to distribute the high traffic
3. Sharded database to handle huge object data
4. Added cache system to reduce load on the database.

# REST APIs
- Create Short Url
- Delete Url
- User Signup, Login, Logout
- Redirect URL

> [!note]
> HTTP **302** Redirect status is sent back to the browser instead of HTTP **301** Redirect. A **301** redirect means that the page has permanently moved to a new location. A **302** redirect means that the move is only temporary. Thus, returning **302** redirect will ensure all requests for redirection reaches to our backend and we can perform analytics (Which is a functional requirement)

# Database Schema

### Short Link

|Field|Type|
|---|---|
|short_url|String|
|original_url|String|
|created_by|Long|
|create_date|DateTime|
|expiry_date|DateTime|

### User

|Field|Type|
|---|---|
|user_id|Long|
|name|String|
|email|String|
|create_date|DateTime|
|password|DateTime|

# Shortening Algorithm