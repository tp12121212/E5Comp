# Sample Glossary of Important Terms for Office 365 Communication Compliance
This project provides a sample glossary for detecting inappropriate workplace communications when using and verifying Office 365 Communication Compliance. The terms are provided as custom confidential information definitions in an [XML](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SIT/CC_Reference.xml) file. Please download and import them into Office 365 using PowerShell. (As of June 2020, keyword detection in Japanese was only possible for Exchange. As of September 2025, Japanese detection works without issue across all services.) Other definitions of personal information for Japan, such as addresses and phone numbers, are introduced on this [page](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SIT.md).

# Run this command once in advance
## Launch PowerShell with administrative privileges and run the following:
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module -Name ExchangeOnlineManagement

# Import sensitive words as custom confidential information in XML format
## Connect to Exchange Online via PowerShell
Import-Module ExchangeOnlineManagement
Connect-IPPSSession

## Upload a local XML file
New-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path "C:\Users\imiki\Desktop\Work\Comp\CC_Reference.xml" -Encoding Byte)

## If you have already uploaded the definition and are updating it to a newer version
Set-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path "C:\Users\imiki\Desktop\Work\Comp\CC_Reference.xml" -Encoding Byte)

# Notes on setting communication compliance policies
Communication When setting and testing compliance policies, please keep the following in mind:
1. When creating a policy targeting multiple sensitive information types, make sure the detection criteria are "any of these," not "all of these."
2. When testing for detection, keep the review rate at 100%.
3. Test detection of a combination of key elements and supporting elements.

In this sample, the confidence level is defined as 60 (low) for detection of only key elements, 80 (high) for detection of key elements and supporting elements, and 70 (medium) for the recommended confidence level.
Unlike DLP policies, Communication Compliance's target confidence level cannot be individually changed; instead, it uses the defined recommended confidence level. Therefore, the policy setting is fixed at medium or higher confidence detection.
As a result, only high-confidence detections of key elements and supporting elements will be hit.

In the XML definition updated on May 15, 2025, the proximity of supporting elements was reduced from 300 to 30, so supporting elements must be present within 30 characters of the main element.

W1. Work Style Words to Watch Out For
S1. Customer Satisfaction Related
S2. Complaint Related
P1. Non-Work-Related Contact
P2. Repeated Dinner Invitations
H1. Power Harassment
H2. Hasty Requests
L1. Illegal
L2. Cartel
Q1. Inspection Falsification
Q2. Foreign Exchange Law
C1. Confidential Conversation

# Defined Forbidden Words
## W1. Work Style Words to Watch Out For
### Content
Terms related to unpaid overtime and excessive working hours.
### Keywords (confidence level 60)
"Unpaid overtime," "Overtime work," "Open holidays," "Unpaid," "Unpaid," "Early morning shifts," "Holiday work," "Labor standards," "Labor standards," "Slavery," "Unpaid work," "All-nighters," "Forced," "Weekends," "Saturdays and Sundays," "Holidays"
### Related words (confidence level 80 when combined with keywords)
"Work," "Duty," "Overtime," "Volunteer work," "Duty"
### Sample
Holidays are also considered work.
Unpaid overtime is work.
Overtime is likely to turn into all-nighters.

## S1. Customer Satisfaction Related
### Content
Terms related to positive reactions, such as gratitude, from customers.
### Keywords (confidence level 60)
"Gratitude," "Like," "Touched," "Thank you," "Great," "Pleasant," "Thank you"
### Related words (confidence level 80 when used in conjunction with keywords)
"Convey," "Customer"
### Sample
We would like to convey our gratitude.
Our customers are pleased.

## S2. Complaint-related
### Content
Terms related to negative reactions, such as customer complaints.
### Keywords (confidence level 60)
"Angry", "Furious", "Indignant", "Complaint", "Grief"
### Related Words (confidence level 80 when combined with keywords)
"Return", "Cancellation", "Leave", "Leave the Room", "Mistake", "Mistake", "Dissatisfaction", "Distrust", "Quality", "Fault", "Problem", "Defect", "Recurrence", "Unexpected", "Dogeza", "Tepidity"
### Sample
There was a <ins>fault</ins> that resulted in a **complaint**.
The customer **angrily** <ins>left the room</ins</ins</ins>.

## P1. Non-Business Communication
### Content
In-house communication and contact, with the understanding that it was non-business.
### Keywords (confidence level 60)
"Off-duty", "Unrelated to work", "On business"
### Related words (confidence level 80 when combined with the keyword)
"Broad", "Rude", "Excuse me", "Sorry", "Discard", "Ignore", "Personal", "Sorry"
### Sample
Excuse me for providing a broad range of information about **off-duty** matters.
This is a **personal** matter unrelated to work, but...

## P2. Dinner Invitations
### Content
Communication regarding invitations to meals or drinks at work.
### Keyword (confidence level 60)
"Drinks", "Dinner", "Meal", "Good Restaurant", "Supper", "Supper", "Drinks"
### Associated Word 1 (confidence level 80 when combined with the keyword)
"Invite", "Not going", "How", "Go", "Shall we go", "Go", "Shall we go?"
### Associated Word 2 (confidence level 90 when combined with the keyword)
"Many times", "Insistent", "Stop", "Stop", "Refuse", "No", "No", "No", "Two people", "Ignore"
### Sample
Would you like to go for drinks?
Please stop persistent invitations to dinner.

## H1. Power Harassment
### Content
Reprimanding, insulting, and aggressive communication within the workplace.
### Keywords (Reliability 60)
"Trash," "Stupid," "Idiot," "Dumb," "Useless," "Incompetent," "Crazy," "Useless," "Die," "God for it," "An eyesore," "Salary thief"
### Related Words (Reliability 80 when used in conjunction with keywords)
"Waste," "Muda," "Muda," "Fired," "Failed," "Demotion," "Common sense," "Obviously"
### Sample
**Incompetent** people should be fired.
**Useless** people should be demoted.

## H2. Hasty Requests
### Content
Communication regarding hasty requests within the company.
### Keywords (Reliability 60)
"Urgent," "Immediately," "Promptly," "Urgently," "Right Away," "Urgent"
### Related Words (Reliability 80 when used with keywords)
"Contact," "Response," "Action," "Answer," "Response," "Reply"
### Sample
Please respond **promptly**.
This is an **urgent** <ins>contact</ins>.

## L1. Illegal
### Content
Communication that refers to violations of the law or crimes.
### Keywords (60 confidence)
"Crime," "Violation of Law," "Illegal," "Unconstitutional," "Illegal," "Violation," "Murder," "Embezzlement," "Break Trust," "Betrayal"
### Related Words (80 confidence when combined with keywords)
"Borderline," "Borderline," "Just barely," "Gray," "Out," "Safe," "Action"
### Sample
**Crime** <ins>Borderline</ins>.
<ins>Just</ins>**Illegal**.

## L2. Cartel
### Content
Communications regarding price and quantity adjustments. Many hits contain content unrelated to cartels, so careful scrutiny is required.
### Keywords (Reliability 60)
"Wholesale price", "Unit price", "Price", "Quantity", "Cartel", "Bid-rigging", "Kickback", "Lobbying"
### Related words (Reliability 80 when combined with keywords)
"Adjustment", "Policy", "Agreement", "Discussion", "Consensus", "Profit", "Margin", "Bidding", "Bidding", "Equivalent product", "Information exchange", "Interview", "Sales", "Wholesale", "Transaction", "Cooperation money", "Contribution money", "Appropriate"
### Sample
We will provide appropriate compensation in return.
Let's discuss the wholesale price.

## Q1. Inspection falsification
### Content
Communication mentioning data falsification or fabrication.
### Keywords (confidence level 60)
"falsification," "fraud," "falsification," "fabrication," "recall"
### Related words (confidence level 80 when combined with keywords)
"non-conformity," "unqualified person," "audit," "quality," "inspection results," "intentional," "deliberate," "overlook," "rewrite," "instructions"
### Sample
The <ins>inspection results</ins> were slightly **falsified**.
We decided to <ins>overlook</ins> to avoid a **recall**.

## Q2. Foreign Exchange Law
### Content
Communication referring to items that can be converted into weapons in accordance with the Foreign Exchange Law.
### Keywords (Reliability 60)
"Weapons", "Weapons of Mass Destruction", "Nuclear", "Superconductivity", "Ceramics", "Machine Tools", "Robots", "Integrated Circuits", "Semiconductors", "Encryption Devices", "Sensors", "Sonar", "Radar", "Lasers", "Navigational Equipment", "Gyroscopes", "Submarines", "Unmanned Vehicles"
### Related Words (Reliability 80 when used with keywords)
"Catchall", "Wassenaar", "Regulation", "Chemical", "Biological", "Missiles", "Rockets", "Nuclear Power", "Foreign Exchange""Diversion", "Overseas", "Foreign Country", "Export", "Exchange Rate"
### Sample
Does this fall under weapons export restrictions?
We are transporting nuclear fuel overseas.

## C1. Confidential Conversation
### Content
Confidential communication within the company.
### Keywords (Reliability 60)
"Just between us," "Can't say it out loud," "Keep it a secret," "Still a secret," "Confidential," "Behind the scenes,"
### Additional words (Reliability 80 with keywords)
"Absolutely," "No matter what," "Must be," "Extremely," "Consideration," "Caution"
### Sample
Please keep this between us.
No matter what, keep it secret.