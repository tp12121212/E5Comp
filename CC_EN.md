# Note sample glossary for Office 365 Communication Compliance
In this project, we provide a sample glossary to detect communication that is not suitable for the workplace when using and verifying Office 365's Communication Compliance. I will do it. The term is [XML](https://github.com/YoshihiroIchinose/E5Comp/blob/main/CC_Reference.xml) as a definition of custom confidential information. It is available in Il, so please download it and import it to Office 365 with PowerShell and use it. As of June 2020, among Exchange/Teams/Skype for Busines/Yammer in Office 365, keyword inspection in Japanese only in Exchange I'm checking the exit. For support of other sources, you need to wait for the update of the Japanese wordbreaker in Office 365. In addition, the definition of personal information such as addresses and phone numbers for Japan is introduced here [page](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SIT.md) I'm doing it.

# Implemented once in advance
## Launch PowerShell with administrative authority and do the following
    Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
    Install-Module -Name ExchangeOnlineManagement

# Impret attention words in XML as custom confidential information
## Connect to Exchange Online from PowerShell
    Import-Module ExchangeOnlineManagement
    Connect-IPPSession

## Upload a local XML file
    New-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path "C:\Users\imiki\Desktop\Work \Comp\CC_Reference.xml" -Encoding Byte)

## If you have uploaded it once and update to the definition of the new version
    Set-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path "C:\Users\imiki\Desktop\Work \Comp\CC_Reference.xml" -Encoding Byte)

# Set the Policy for Communication Compliance
Set the Communication Compliance policy to detect communications that contain the following custom sensitive information imported. In the case of detection of only each keyword, the accuracy is 60 or more, and if the detection including incidental words is targeted, the accuracy is set to 80 or more.

W1. Work style caution word
S1. Customer satisfaction related
S2. Complaint-related
P1. Contact outside of work
P2. Invitation to repeated meals
H1. Power harassment
H2. Urgent request
L1. Illegal
L2. A cartel
Q1. Inspection disguise
Q2. Foreign exchange law
C1. A confidential talk

# Predefined NG words
## W1. Work style caution word
### Contents
Terms related to service overtime and excessive labor.

### Keyword (Reliability 60)
"Service overtime", "rust residue", "no holiday", "unpaid", "free", "early morning attendance", "holiday attendance", "labor base", "labor standards", "slave", "just work", "all night", "forced", "weekends", "Saturdays and Sundays", "holidays"

### Accompanying words (reliability 80 as a set with keywords)
"Work", "Work", "Overtime", "Volunteer", "Work"

## S1. Customer satisfaction related
### Contents
Terms related to positive reactions such as gratitude from customers.

### Keyword (Reliability 60)
"Thank you", "Like", "I'm impressed", "Thank you", "Good", "Pleased", "Thank you"

### Accompanying words (reliability 80 as a set with keywords)
"Tell", "Customer"

## S2. Complaint-related
### Contents
Terms related to negative reactions such as customer complaints.

### Keyword (Reliability 60)
"Angry","Fury","Indignation","Creaint","Complaint"

### Accompanying words (reliability 80 as a set with keywords)
"Return", "Cancellation", "Exit","Exit","Mistake","Mistake","Dissatisfaction","Distrust","Quality","Fault","Disability","Defect","Recurrence","Unexpected","Doltdown","Temper"

## P1. Contact outside of work
### Contents
Contact and communication within the company after refusing outside of work.

### Keyword (Reliability 60)
"Outside work", "It has nothing to do with work", "In business"

### Accompanying words (reliability 80 as a set with keywords)
"Extensive", "Excuse me", "I'm sorry", "I'm sorry", "Discarded", "Ignore", "Personal", "I'm sorry"

## P2. Invitation to a meal
### Contents
Communication on invitations to eat and drink at work.

### Keyword (Reliability 60)
"Drink","Dinner","Dinner","Meal","Good restaurant","Dinner","Dinner","cup"

### Accompanying word 1 (Reliability 80 with keyword and set)
"Invitation", "Don't go", "How", "Go", "Would you like to go", "Go", "Would you like to go"

### Accompanying word 2 (reliability 90 as a set with keywords)
"Many times", "Insistance", "Stop", "Stop", "No", "No", "No", "No", "No", "Two people", "Ignore"

## H1. Power harassment
### Contents
Reprimands, insults and offensive communication within the company.

### Keyword (Reliability 60)
"Scum", "stupid", "stupid", "stupid", "stupid", "useless", "incompetent", "crazy", "unusable", "die", "die", "an eyesore", "pay thief"

### Accompanying words (reliability 80 as a set with keywords)
"Waste", "Waste", "Waste", "Kubi", "Kubi", "Disqualified", "Demotion", "Common sense", "Natural"

## H2. Urgent request
### Contents
Communication on urgent requests within the company.

### Keyword (Reliability 60)
"Urgent", "immediately", "immediately", "immediately", "immediately", "immediately"

### Accompanying words (reliability 80 as a set with keywords)
"Contact", "Response", "Treatment", "Answer", "Response", "Reply"

## L1. Illegal
### Contents
Communication that refers to violations of the law and crimes.

### Keyword (Reliability 60)
"Crime", "violation of the law", "illegal", "unconstitutional", "illegal", "violation", "murder", "embezzlement", "betrayal", "betrayal"

### Accompanying words (reliability 80 as a set with keywords)
"Suresure","Suresure","Giri","Gray","Out","Safe","Acts"

## L2. A cartel
### Contents
Communication on price and quantity adjustment. Many contents that are not related to the cartel also hit, so it needs to be scrutinized.

### Keyword (Reliability 60)
"Wholesale price", "Unit price", "Price", "Quantity", "Cartel", "Ranguing", "Return", "Talking"

### Accompanying words (reliability 80 as a set with keywords)
"Adjustment", "Policy","Agreement","Consultation","Agreement","Profit","Interest","Bid","Bid","Equivalent","Information Exchange","Interview","Sales","Wholesale","Transaction","Cooperation Money","Approval","Commensurate"

## Q1. Inspection disguise
### Contents
Communication that refers to data falsification and deception.

### Keyword (Reliability 60)
"Fake", "Fraud", "Falsification", "Falsification", "Fake", "Recall"

### Accompanying words (reliability 80 as a set with keywords)
"Nonconformity", "Unqualified person", "Audit", "Quality", "Inspection result", "Intentional", "Intentional", "Forest", "Rewrite", "Instruction"

## Q2. Foreign exchange law
### Contents
Communication that refers to goods that can be diverted into weapons consistent with the Foreign Exchange Act.

### Keyword (Reliability 60)
"Weapons", "Weapons", "Mass destruction", "Nuclear", "Superconductivity", "Ceramic", "Machine tools", "Robots", "Integrated circuits", "Semiconductors", "Cryptographic devices", "Sensors", "Sonar", "Radar", "Laser", "Nutramation Place","Gyroscope","Submersible","Unmanned"

### Accompanying words (reliability 80 as a set with keywords)
"Catchall", "Wassener","Regulation","Chemistry","Biology","Missile","Rocket","Nuclear Power","Forex","Diversion","Overseas","Foreign","Export","Exchange"

## C1. A confidential talk
### Contents
In-house confidential communication.

### Keyword (Reliability 60)
"Just between you and me", "I can't say it out loud", "Keep it a secret", "It's still a secret", "Confidentially", "Out of the back"

### Accompanying words (reliability 80 as a set with keywords)
"Absolutely","No matter what happens", "Absolutely", "Be careful", "Consideration", "Caution"