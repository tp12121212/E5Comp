# Microsoft Purview Communication Compliance, Sample Glossary of Important Terms 2
This project provides a sample glossary for detecting inappropriate communications when using and verifying Microsoft Purview's Communication Compliance.
The terms are provided as custom confidential information definitions in the [XML](https://github.com/YoshihiroIchinose/E5Comp/blob/main/SIT/StockManipulationJP.xml) file.
Please download it and import it into Office 365 using PowerShell. For glossary 1, please refer to this [page](https://github.com/YoshihiroIchinose/E5Comp/blob/main/CC.md).

## Keyword Creation Method
CC is gradually implementing pre-trained trainable classifiers and LLM-based detection using Azure Content Safety, which is currently available in preview as of August 2024, to identify inappropriate communications and images for the workplace.
However, custom trainable classifiers and pre-trained Stock Manipulation trainable classifiers do not currently support Japanese.
Therefore, this sample uses Copilot for Microsoft 365, a generative AI, to output sample messages related to market manipulation and stock price manipulation.
To detect these messages, characteristic keywords from the messages were extracted and applied to the confidential information type settings available in Purview.
The Purview settings are configured to include a list of keywords that can be matched by string, allowing detection independent of word breaking.
Furthermore, for keywords that are unlikely to be hit universally, false positives are suppressed by requiring supporting keywords within 30 characters before and after the keyword.
When deploying in a real environment, please further suppress false positives, add detection scenarios, and refine the detection as necessary.

# Run this once in advance
## Launch PowerShell with administrative privileges and run the following:
```PowerShell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
Install-Module -Name ExchangeOnlineManagement
```

# Import sensitive words as custom confidential information in XML
## Connect to Exchange Online from PowerShell
```PowerShell
Connect-IPPSSession
```
## Upload a local XML file
```PowerShell
New-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path "C:\Users\imiki\Desktop\Work\Comp\StockManipulationJP.xml" -Encoding Byte)
```
## If you have already uploaded the definition and are updating it to a newer version
```PowerShell
Set-DlpSensitiveInformationTypeRulePackage -FileData (Get-Content -Path "C:\Users\imiki\Desktop\Work\Comp\StockManipulationJP.xml" -Encoding Byte)
```
# Configure Communication Compliance Policy
Configure Communication Compliance policy to detect communications containing the following custom confidential information.

M1. Market Manipulation

# Predefined NG Words
## M1. Market Manipulation
This sample detects messages suggesting market manipulation or stock price manipulation.
### Pattern 1. Market Manipulation Single Prohibited Words
#### Key Keywords (High Confidence)
```
Dummy Trading
Conformity Trading
High Price Creation
Low Price Creation
Price Manipulation
Insider Information
Spurious Trading
Proprietary Trading
Market Manipulation
Stock Price Manipulation
Price Manipulation
Price Manipulation
Selling Down
Spurious Order
```
### Pattern 2. Market Manipulation Prohibited Actions
#### Key Keywords
```
Buying in Large Quantities
Selling in Large Quantities
Selling in Small Quantities
Selling in Small Quantities
Selling in Small Quantities
Buying in Large Quantities
Selling in Large ...Concentrated Trading
Buying in Large Quantities
Selling in Large Quantities
Concentrated Trading
Concentrated Orders
Concentrated Trading
Concentrated Trading
Misleading
Buying in Large Quantities
Selling in Large Quantities
Cancelling Orders
Cancelling Orders
Cancelling Orders
Closing Price
Buying in Large Quantities
Selling in Large Quantities
Concentrated Trading
Spreading Bad Rumors
```
#### 30 Items Before and After the Key Quantities Required auxiliary elements within a character (detected with high confidence only when combined with the main element)
```
Raise
Lower
Raise
Lower
Decrease
Increase
High
Low
Lower
Lower
Fix
Stable
Move the market
Raise the market
Lower the market
Raise the stock price
Lower the stock price
Raise the price
Lower the price
Favorable trading
Unfavorable trading
Raise
Lower
Raise
Lower
Specific time
Specific price
Attract attention
Look
First
Attract
Make people think
```

### Pattern 3. Market manipulation and false information behavior
#### Key element keywords
```
Lead
Mislead
Confuse
Suppress
Pressure
Relieve
Conduct trading
Use information
Adjust timing
Increase trading volume
Decrease trading volume
Utilize
Actual trading
Actual buying and selling
Conclude contract
Conflict of interest
Complete transaction
Buy stocks
Sell stocks
```
#### 30 before and after the main element Required auxiliary elements within a character (detected with high confidence only when combined with the main element)
```
False
Different
Insider
Internal
Top Secret
Secret
Confidential
Confidential
Confidential
Undisclosed
Undisclosed
Accusation
False
Competitor
News
On social media
Before earnings announcement
Not aware
Not aware
In the market
Competitor
Bribery
Involved
Tampering
Falsification
Certain
Before
Expected
Planned
```

# Sample message generated by Copilot for Microsoft 365 suggesting market manipulation to be detected
Main element - bold
Auxiliary element - underline
### Pattern 1.
Create a **fake order** to confuse other traders
Increase trading volume by **washing trades**
Manipulate prices by **collaborating**
Depress stock prices by **selling off**
Place consecutive buy orders to **create a high price**
Place continuous sell orders to **create a low price**

Spread specific news to **move stock prices**

Use insider information** to trade to your advantage

Collaborate with other traders to **manipulate the market**

Engage in proprietary trading to increase trading volume

Spread specific news to **move stock prices**

Spread rumors on social media to **manipulate stock prices**

Place sham orders to lure other investors in

### Pattern 2.
Repeatedly buy and sell at a specific price range to fix the price
Place large buy orders to raise the stock price
Sell gradually to reduce selling pressure
Concentrate buy orders in the last 10 minutes to increase the closing price
Buy up specific stocks to attract other investors
Concentrate your trading during a specific time period.
Cancel orders to reduce market liquidity.
Place orders at a specific price range to attract attention.
Focus on trading specific stocks to attract attention.
Watch the movements of other traders and go against them.
Repeatedly buy and sell within a specific price range to stabilize prices.
Buy and sell specific stocks to raise prices.
Sell and sell specific stocks to lower prices.
Concentrate your trading during a specific time period.
Buy large quantities of specific stocks to raise prices in the next trade.
Spread rumors that Company B is releasing a new product to raise its stock price.
Spread negative rumors to lower a competitor's stock price.
Place a large buy order for Company A's stock this week to make other investors believe there is active trading activity.

### Pattern 3.
Spread fake news to mislead the market.
Use this information to trade before next week's earnings announcement.
Time your trades so other investors don't notice.
Increase trading volume to mislead the market.
Report transactions that differ from actual trading.
Gather trading information from your competitors to use in your strategy.
Put pressure on your employees to prevent whistleblowing.
It's okay to give a small bribe to get this contract.
Don't tell your boss you're involved in this deal. It could pose a conflict of interest.
They falsified the documents to close the deal.
The project is sure to be a success, so it's a good idea to buy some stock now.
Before the management changes are announced, it's a good idea to buy some stock.
The financial situation is expected to improve, so it's a good idea to buy some stock before the stock price rises.
The company's debt is expected to be reduced, so it's a good idea to buy some stock now.
A new CEO is expected to be installed, so it's a good idea to buy some stock before the stock price rises.
Before the new partnership is announced, it's a good idea to buy some stock.
The new project is progressing smoothly, so it's a good idea to buy some stock before the stock price rises.
A contract with a major company will be signed soon, so buy some stock before the price goes up.