<!-- @format -->

# Smart Static Website Hosting on AWS

**Technologies**: S3 (Private), CloudFront (OAI), AWS WAF, CloudWatch Logs, SNS Notifications.

**Focus:** Secure, Monitored, Scalable, Serverless Hosting within AWS

## 📄 Project Objective

The goal of this project is to securely host a static website on AWS without making the S3 bucket public. The architecture leverages CloudFront OAI (Origin Access Identity) to securely serve the content, WAF to protect against common web attacks, CloudWatch Logs for monitoring & SNS Notifications for alerting on potential threats — all carefully designed to remain within AWS Free Tier usage limits.

## 🔧 **AWS Services Used**

<div align="center">

| **Service**                      | **Purpose**                                       |
| -------------------------------- | ------------------------------------------------- |
| **Amazon S3**                    | Host static website files (Private Bucket)        |
| **CloudFront**                   | CDN with HTTPS and OAI for private S3 access      |
| **OAI (Origin Access Identity)** | Restrict direct S3 access, grants only CloudFront |
| **AWS WAF**                      | Web security: blocks malicious traffic            |
| **CloudWatch Logs**              | Logs WAF traffic                                  |
| **CloudWatch Alarms**            | Trigger alerts for security events                |
| **SNS**                          | Email alerts for suspicious traffic               |

---

</div>

## 📂 Architecture Diagram

  <div align="center">
      <img src="Diagrams/WorkFlow.png" width=100%>
  </div>
  
### 📋 Architecture Summary
This project delivers a secure static website hosting solution on AWS Free Tier using the following architecture:
- **Amazon S3 (Private)** stores the website content.
- **CloudFront (HTTPS, OAI)** securely serves the content without exposing the S3 bucket publicly.
- **AWS WAF** protects the application from common web threats (SQLi, XSS, IP reputation, bots) using AWS Managed Rule Sets.
- **CloudWatch Logs** captures WAF logs for monitoring.
- **CloudWatch Alarms** trigger on suspicious activity (blocked requests).
- **SNS (Email)** notifies the security team of any potential attacks.

## 📋 Detailed Step-by-Step Setup

### ✅ Step 1: Create a Private S3 Bucket

- **Actions:**

  - Create an S3 bucket.
  - Enable **Block All Public Access** to keep the bucket private.

    <div align="center">
        <img src="Diagrams/S3-Block-Public-Access.png" width=100%>
    </div>

  - Upload your static website files (e.g., `index.html`, `error.html`).
  - **Do NOT** create a public bucket policy.
  - Confirm direct access via the S3 URL results in **Access Denied**.

- **Troubleshooting:**
  - **Problem:** Confused due to absence of bucket policy.
  - **Solution:** CloudFront with OAI handles the permissions, no public access is required for the S3 bucket.

---

### ✅ **Step 2: Create CloudFront Distribution with OAI**

- **Actions:**

  - Set **Origin Domain** as: **`bucket-name.s3.<region>.amazonaws.com`**.
  - Allow private S3 bucket access to CloudFront.

    <div align="center">
        <img src="Diagrams/Allow-private-S3-bucket-access-to-CloudFront.png" width=100%>
    </div>

  - Set **Viewer Protocol Policy** to **Redirect HTTP to HTTPS**.
  - Set **Default Root Object** to `index.html`.

    <div align="center">
        <img src="Diagrams/Default-Root-Object.png" width=100%>
    </div>

  - Deploy the Distribution & wait until deployment is complete.

- **Access the site via :**

  ```url
  https://<distribution-id>.cloudfront.net/index.html
  ```

- **Troubleshooting:**

  - **Problem:** Seeing **“Access Denied”** error when accessing the CloudFront URL.
  - **Solution:**

    1. **Confirm OAI permissions:** Ensure that the Origin Access Identity (OAI) is properly configured & associated with your CloudFront distribution. AWS automatically applies the necessary permissions to allow CloudFront access to the S3 bucket.
    2. **Ensure you are not accessing the S3 website endpoint:** Make sure you are using the CloudFront distribution URL not the S3 website URL as the latter does not have the proper permissions set for CloudFront.
    3. **Don't forget to set **`index.html`** as the default root object:** If you miss setting **`index.html`** as the root object in CloudFront you will encounter an Access Denied error because CloudFront won't know which file to load by default.
    4. **Cache Invalidation after adding the root file:** After you add or update the **`index.html`** (or any other file) in your S3 bucket you may need to invalidate the CloudFront cache to reflect the changes. If the old cached version is still served you may continue seeing errors or outdated content. Invalidate the cache via the CloudFront console to ensure the new files are properly loaded.

       <div align="center">
           <img src="Diagrams/Cache-Invalidation.png" width=100%>
       </div>

       <div align="center">
           <img src="Diagrams/Cache-Invalidation01.png" width=100%>
       </div>

---

### ✅ **Step 3: AWS WAF Setup**

- **Actions:**

  - Create a **Web ACL**.
  - Set **Resource Type** to **Global resources**.

    <div align="center">
    <img src="Diagrams/Global-Resource.png" width=100%>
    </div>

  - Provide a **Name** for your Web ACL.
  - Under **Associate Resources** select **CloudFront** & then choose your **CloudFront distribution** from the available resources.

    <div align="center">
    <img src="Diagrams/Add-Resource01.png" width=100%>
    </div>

- **Add the following AWS Managed Rule Groups:**

  - AWS offers two types of rule groups: **free** & **paid**. For this project, we will select the **free** rule groups based on our requirements.
  - Add the following **AWS Managed Rule Groups**:
    1. **AWS-AWSManagedRulesAdminProtectionRuleSet**
    2. **AWS-AWSManagedRulesAmazonIpReputationList**
    3. **AWS-AWSManagedRulesAnonymousIpList**
    4. **AWS-AWSManagedRulesKnownBadInputsRuleSet**
    5. **AWS-AWSManagedRulesCommonRuleSet**
  - In the **Priority** field, remember that a **lower number** indicates **higher priority**. So, assign priority numbers accordingly.

    <div align="center">
    <img src="Diagrams/Rules03.png" width=100%>
    </div>

  - **Recommended Priority:**

    1. AdminProtection
    2. AmazonIPReputation
    3. AnonymousIPList
    4. KnownBadInputs
    5. CommonRuleSet

  - For each rule in the AWS Managed Rule Groups, change the action from **Count** to **Block**. This ensures that any malicious traffic matching these rules is actively blocked rather than just counted.

    <div align="center">
    <img src="Diagrams/Rules04.png" width=100%>
    </div>

- **Troubleshooting:**

  - **Problem:** Uncertainty about WAF functionality.
  - **Solution:** Test with malicious paths (e.g., **`/admin`**) or SQL injection payloads and confirm that WAF logs are showing up in CloudWatch.

---

### ✅ **Step 4: CloudWatch Logs Integration (WAF Logging)**

- **Actions:**

  - Create a **CloudWatch Log Group**:
    - Name it: `aws-waf-logs-smart-static-website` (Note: The naming convention is **critical**).
  - Go to **WAF & Shield** in the AWS console then navigate to your **Web ACL**.
  - Under **Logging and Monitoring** enable **Logging**.

    <div align="center">
    <img src="Diagrams/Enable-Logging.png" width=100%>
    </div>

  - Set the destination to **CloudWatch Logs**.
  - Choose the log group you created.

- **Troubleshooting:**
  - **Problem:** Log group isn’t visible in WAF.
  - **Solution:** Ensure the log group name follows the **`aws-waf-logs-`** naming convention.

---

### ✅ **Step 5: SNS Email Notification**

1.  **Create an SNS Topic:**

    - Navigate to **SNS** in the AWS Console.
    - Select **Create Topic**.
    - Choose **Standard** as the topic type.

     <div align="center">
     <img src="Diagrams/SNS.png" width=100%>
     </div>

    - Give your topic a name, for example, `waf-alerts-topic`.
    - Click **Create Topic**.

2.  **Subscribe Your Email to the SNS Topic:**

    - After creating the SNS topic select the topic name (`waf-alerts-topic`).
    - Under **Subscriptions**, click **Create subscription**.

       <div align="center">
       <img src="Diagrams/SNS01.png" width=100%>
       </div>

    - Choose **Protocol** as **Email**.

      <div align="center">
      <img src="Diagrams/SNS02.png" width=100%>
      </div>

    - Enter your email address and click **Create subscription**.

      <div align="center">
      <img src="Diagrams/SNS03.png" width=100%>
      </div>

      - You will receive a confirmation email. **Click the confirmation link** to activate the subscription.

      <div align="center">
      <img src="Diagrams/SNS04.png" width=100%>
      </div>

      <div align="center">
      <img src="Diagrams/SNS05.png" width=100%>
      </div>

---

### ✅ **Step 6: Create CloudWatch Alarm**

1. **Navigate to CloudWatch Alarms:**

   - Go to **CloudWatch** and select **Alarms**.
   - Click on **Create Alarm**.

2. **Choose Metric for WAF Blocked Requests:**

   - Select the **WAFV2** namespace.
   - Choose the **BlockedRequests** metric for your **Web ACL**.

3. **Configure the Alarm:**

   - Set the **Period** to **1 minute**.
   - Set the **Threshold** to `>= 1 blocked request`.
   - Choose the **Statistic** as **Sum** and **Evaluation Periods** as `1`.

4. **Attach the SNS Topic:**

   - In the **Actions** section, choose **Send a notification**.
   - In the **Send notification to** dropdown, select the **SNS Topic** you created earlier (`waf-alerts-topic`).

5. **Save the Alarm:**
   - Click **Create Alarm** to save the alarm.

---

### ✅ **Step 7: Result: Test and Verification**

1. **Website Screenshot with **`/admin`** URL:**

   - When you visit the `/admin` URL (which is intentionally blocked by WAF), the request should be blocked & you should see an **Access Denied** page. Below is a screenshot showing the website behavior when trying to access the `/admin` path.

    <div align="center">
    <img src="Diagrams/Result.png" width=100%>
    </div>

2. **Traffic Statistics:**

   - Below is a screenshot showing the action totals for all traffic processed by AWS WAF during the test period. This includes the number of blocked requests, allowed requests, and the percentage distribution.

      <div align="center">
      <img src="Diagrams/Result02.png" width=100%>
      </div>

3. **CloudWatch Logs:**

   - After triggering the blocked request (by visiting `/admin`), the blocked request will be logged in **CloudWatch Logs**. Here’s an example of how the log entry should look like in CloudWatch:

    <div align="center">
    <img src="Diagrams/Result05.png" width=100%>
    </div>

4. **CloudWatch Alarm State:**

   - After the request is blocked, **CloudWatch Alarm** will be triggered. The alarm state will change to **ALARM**, and it will send a notification through **SNS**. Below is an example of the alarm state:

     - **Alarm State:** `ALARM`
     - **Metric:** BlockedRequests
     - **Threshold:** >= 1 blocked request
     - **Period:** 1 minute

        <div align="center">
        <img src="Diagrams/Result06.png" width=100%>
        </div>

5. **Email Notification:**

   - When the **CloudWatch Alarm** is triggered an email notification will be sent via the **SNS Topic**. The email will contain details of the blocked request and alert you that the WAF has blocked suspicious traffic.

   <div align="center">
   <img src="Diagrams/Result03.png" width=100%>
   </div>

## ⚠️ **Common Mistakes & Lessons Learned**

<div align="center">

| **Mistake**                               | **Lesson**                                  |
| ----------------------------------------- | ------------------------------------------- |
| Not following **`aws-waf-logs-*`** naming | WAF won’t recognize invalid log group names |
| Forgetting Default Root Object            | Causes Access Denied on `/` root URL        |
| Expecting S3 access without OAI           | OAI is mandatory for private S3 access      |
| Using the wrong S3 endpoint (s3-website)  | Use REST endpoint for CloudFront access     |
| Not confirming SNS email subscription     | Alarm won’t notify until confirmed          |

</div>

---

## 🔐 **Security Best Practices Followed**

<div align="center">

| **Component**  | **Security Mechanism**              |
| -------------- | ----------------------------------- |
| **S3 Bucket**  | Private, no public access, OAI only |
| **CloudFront** | OAI secured, HTTPS enforced         |
| **WAF**        | Blocks known threats                |
| **CloudWatch** | Logs security events                |
| **SNS**        | Alerts via email                    |

</div>

---

## 🔥 Project Outcome

**1.** Fully secured static website hosted on AWS Free Tier.

**2.** Global delivery via CloudFront with HTTPS enforcement.

**3.** Web security protections via AWS WAF.

**4.** Logs stored for auditing in CloudWatch Logs.

**5.** Real-time alerts via SNS notifications.

**6.** No direct public access to S3.

---

## 🚮 Cleanup Steps to Avoid Charges

**1.** Delete CloudFront Distribution (wait until fully disabled).

**2.** Delete S3 Bucket.

**3.** Delete WAF Web ACL.

**4.** Delete CloudWatch Log Group: aws-waf-logs-smart-static-website.

**5.** Delete CloudWatch Alarms.

**6.** Delete SNS Topic and Subscriptions.
