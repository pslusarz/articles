---
title: "Comparing Tesseract and GPT4 Optical Character Recognition on NARA Rolls"
date: 2023-12-22
---

"NARA Rolls" is a colloquial name given to a <a href="https://www.archives.gov/research/captured-german-records">collection of captured German World War 2 documents</a> that were microfilmed in the 50s. Here I'm working on T77 roll 619, which I obtained as scanned jpgs after paying National Archives around $130. Just this one roll contains over 1,000 pages, and the entire dataset is approximately 70,000 rolls. This dataset contains a very detailed record of the operations of the Nazi Germany and while technically available to the public, it has been difficult to use in research due to the above described method for obtaining the data, and the format in which it comes. To make things even harder, I don't even know German. However, I think that with the modern technologies, it should be fairly feasible to build this into a multi lingual searchable dataset that can be added to over time.

Not sure where to begin, I'd like to start by comparing two OCR technologies: Tesseract and GPT4, which recently aquired "vision" capabilities - meaning you can submit an image to it and ask it questions about the image. While I try to be methodical, I realize that this is just a snapshot in time ofthe capabilities of each of these technologies, and so the main benefit of this project is to obtain insights about the data and challenges with OCR. With these insights, I hope to be able to better approach the rest of this project.

For comparison, I selected 25 random files from the dataset of over 1000. When the random number pointed to a file that was not a good candidate for OCR, like a separator title page, I advanced to the next one that might contain a useful text. I may have done it twice or so.  

The raw data is as follows:

| NARA page scan | GPT4-vision-preview | Tesseract 5.3.3 |
| -------------- | ------------------- | --------------- |
| [0028.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0028.jpg) | | |
| [0053.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0053.jpg) | | |
| [0129.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0129.jpg) | | |
| [0197.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0197.jpg) | | |
| [0200.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0200.jpg) | | |
| [0235.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0235.jpg) | | |
| [0268.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0268.jpg) | | |
| [0305.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0305.jpg) | | |
| [0360.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0360.jpg) | | |
| [0408.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0408.jpg) | | |
| [0413.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0413.jpg) | | |
| [0425.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0425.jpg) | | |
| [0499.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0499.jpg) | | |
| [0598.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0598.jpg) | | |
| [0606.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0606.jpg) | | |
| [0608.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0608.jpg) | | |
| [0612.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0612.jpg) | | |
| [0667.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0667.jpg) | | |
| [0730.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0730.jpg) | | |
| [0820.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0820.jpg) | | |
| [0836.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0836.jpg) | | |
| [0839.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0839.jpg) | | |
| [0894.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0894.jpg) | | |
| [0931.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/0931.jpg) | | |
| [1042.jpg](/articles/assets/2023-12-22-compare-ocr-tesseract-gpt4-nara-rolls/raw/1042.jpg) | | |

