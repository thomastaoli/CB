# Community Boards in New York
Examining meeting records from all Community Boards in New York City presents a formidable challenge due to the significant inconsistency in data formats across the boards. This study specifically targeted Manhattan's Community Board 1 (CB1), renowned for its well-digitalized and comprehensive record-keeping. The initial step involved scraping the CB1 website to retrieve categorized files, which included meeting minutes, agendas, and voting records, all in PDF format.

## Data Collection Process

I got CB level ACS data from census reporter, including income, population, poverty and age. I also scraped CB1 meeting minutes from its web page.

## Cleaning
Subsequently, the files were converted into a text format using pdfminer, a tool adept at extracting text from PDF files. To structure and analyze the textual data effectively, the OpenAI API was employed with three distinct prompts designed to extract and synthesize key information.

## Data Analysis
I used following prompts to turn text into structured data:
Key Terms: Prompted to "Return two key terms from the meeting minutes," aiming to highlight the primary topics of discussion.
Compliance: Asked, "Did the meeting comply with the Open Meetings Law and NYC Charter?" This prompt required a simple "Yes" for compliance, or, if non-compliant, a brief explanation of the reasons.
Interesting Aspects: Sought insights on "two key terms about anything noticed as interesting, unusual, or unexpected for a community board meeting," which helped identify unique or noteworthy elements within the meetings.
