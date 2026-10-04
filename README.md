# I Only Wanted a RAG for Mon, and Ended Up Building an OCR

_Mon shares its script with Burmese, so the first models I tried read it as Burmese. This is how a small RAG turned into an OCR, and what I learned about text on the way._

![Article cover - A page of text lines beside a grid of stored characters, with one row out of line](./assets/i-only-wanted-a-rag-for-mon.avif)

I wanted a small RAG for Mon, my native language. The idea was simple: collect some Mon text, make
it searchable, and ask it questions. I thought it would take a weekend.

It's taken more than a year. The first models I tried read Mon as Burmese, which makes sense once
you know the history.

Mon is an ancient language of Southeast Asia, spoken for well over a thousand years in what is now
Myanmar and Thailand, with a tradition that reaches back more than 2,500 years. The [last Mon
kingdom](https://en.wikipedia.org/wiki/Mon_kingdoms) fell in 1757, and the language has lost ground
since. UNESCO's 2010 atlas listed it as vulnerable.

Mon is also widely held to be the source of the Burmese script. In the traditional account, the
Burmese king Anawrahta conquered the Mon city of Thaton in the 11th century, and the Burmese adapted
the Mon script to write their own language. The two languages aren't related, but the scripts are
close relatives. Mon still has letters that Burmese doesn't, so a model trained only on Burmese has
nothing to say when it meets them.

So there wasn't much to start from. I couldn't find a tokenizer, a language detector or a dataset
for Mon. I decided to make them, the NLP tools first and language models later. Every one of them
needed text.

## Finding the text

I took whatever I could reach: Mon Wikipedia, a news site, posts from Facebook and Telegram, a
dictionary database, and a few smaller sets. I put it all in one public collection,
[MonCorpusCollection](https://github.com/MonDevHub/MonCorpusCollection). It came to about 47 million
characters, mostly from Wikipedia and the news site. That's enough to build tools on, and far too
little to train a language model from scratch.

The text I wanted most was in books, and most of the books were scans. A PDF of a page image is easy
for a person to read and useless to a search index. To use them, I first had to turn them into text.

## Reading the pages, on a phone

That meant OCR. I looked at what already existed first: Kraken, Tesseract, TrOCR and PaddleOCR. In
September 2025 I set up Kraken and a TrOCR fine-tuning script next to a small recogniser of my own.
Mine is the one that stayed. I haven't run a head-to-head comparison on Mon pages yet, so I won't
claim how it ranks. That test is on my list.

The design came from who I expected to use it. Most people who'd want to read these books are on
phones, and many are in rural areas with a poor connection. A service that needs you to upload a
whole book doesn't help them. So I built the mobile model first, small enough to run on the device
and work offline. A bigger server model could come later.

## Three ways to write the same letters

Later I went back to the books that did have a text layer, hoping to learn what was inside. I
checked thirteen, and one of them extracts like this:

`k[.j}tY`

On screen, that page is Mon. In the file it's plain ASCII. The book uses an old font that draws Mon
letters on the slots meant for English ones, so the Mon only exists while that font is drawing it.

Myanmar script has been written down in at least three ways. Unicode gives every letter one agreed
number. Zawgyi was the usual way to type Burmese for years. It uses the same numbers but draws some
of them as different letters, so text looks right in one font and wrong in another. And old fonts
like this one ignore the numbers altogether. Unicode is recent in Myanmar publishing, and for Mon it
was rare. Of the thirteen, ten were legacy fonts and three were Zawgyi. None was Unicode.

Converting them wasn't an option either. Rabbit, the standard Zawgyi converter, is written for
Burmese, and I couldn't find a reviewed one that handles Mon. Mon uses eleven characters that
Burmese doesn't. They're a small share of any page, but about 60% of the Mon lines I counted use at
least one.

Then I saw what should have been obvious. The OCR doesn't care. The model reads pixels, so it never
sees the encoding. That legacy-font book is one of the cleanest results I have. The text layers were
no use to me, and the page images were fine.

## How the model got here

The first recogniser was a basic CRNN, a small convolutional network with an LSTM on top. The first
serious one used ResNet-18 and read lines 64 pixels high. Mon stacks vowel signs and small marks
above and below each letter, and at that height they blurred together. The next version moved to
MobileNetV3 with taller lines, but it still struggled with the denser combinations and had no
attention layer.

I also built a larger server design, a Swin transformer with an autoregressive decoder, and archived
it before it finished training. It was heavy and a different kind of model to maintain. It also
worked against the whole point: a model that runs on a phone.

The one I kept closest was the SVTR-style recogniser in PaddleOCR's mobile pipeline. It's a good
design, and it was built for phones. I didn't switch because it would have meant throwing away the
export and checks I'd already built for every platform. It's still the first alternative I'd go back
to.

I also passed on decoders that build in a language model. I wanted the output to come from the
image, not from guesses about what Mon usually says. That has a cost. When a letter is smudged, the
model can't lean on the words around it. I'd rather see that error than have a model quietly cover
it with a plausible Mon word.

The current model, v3.5, is MobileNetV3-Large with two BiLSTM layers, a small attention block and a
CTC head. It has about 11.5 million parameters, and the exported ONNX file is about 46 MB.

## The bug that looked like the model

The hardest bug wasn't in any of those models. I trained on synthetic lines: render text in a font,
and use that text as the label. One font, UniMon, turned out to be Zawgyi-encoded. It used the right
Unicode numbers but drew several of them as the wrong letters. My validation lines, the ones I held
back to score the model while it trained, used only that font. So for a while the scores marked the
model wrong for reading the image correctly.

It was the same problem as those text layers, only this time it was in my own data.

When I found it, I wrote a check for exactly this. For a day the data generator still didn't call
it, while a document said the checks were in place. The code said otherwise. Now every font has to
pass one rule before it's used. It has to be real Unicode, not Zawgyi in disguise, and it must
include the Mon letters. The data generator and the audit run the same test.

## Where it is now

Today you can open the [web app](https://ocr.mondevhub.com), drop in a page, and get Unicode Mon
back, with nothing uploaded. The same model runs in Android and iOS apps, which you build from
source for now, and in a command-line tool that reads whole PDFs and folders of images in batch.

It reads PDFs in any of the three encodings and screenshots, and in my own use it handles photos I
take on my phone and posters too. In three published samples, a Zawgyi PDF, a legacy-font PDF and a
typeset screenshot, none of the 563 lines came out garbled. I picked those three from a wider
screening, so they show it at its best. Across a bigger pile of books, about 9% of lines come out
garbled.

On 150 held-out lines, in a typeface the model never saw, about one character in a hundred is wrong.
The character error rate is 0.0100, with a 95% interval of 0.0056 to 0.0147. Those lines are
synthetic, and I haven't put a number on photographs yet. The [model
card](https://huggingface.co/janakhpon/monocr) has the details.

The command-line tool is also what brought me back to where I started. I've been using it to read
books from my archive into the same collection, a page at a time. There are 16 texts in it so far.
Each one has a record of its source and of which pages were kept. Pages that come out empty or
garbled are dropped, and nobody has proofread the rest yet.

## What's next

MonOCR is still a work in progress. I work on it on weekends, fixing what breaks and improving
what's there, and the corpus grows a book at a time. Next I want a reviewed set of real Mon pages,
so I can measure accuracy where it matters and compare the model with Kraken, Tesseract and
PaddleOCR properly. I still want a bigger server model for the hard pages, alongside the phone one.
Its training pipeline is already built. And seven of the nine sources in the collection don't have
an established licence yet, so I'm careful about what I call reusable.

Looking back, most of the work wasn't training a model. It was finding out what the text really was
before I trusted it: what a font drew, what an encoding meant and what a score was measuring. If I
started again, I'd check the data and the fonts before training anything.

The RAG is still unfinished. But it finally has something to search. Books that were only page
images last September are text now, and the tools that read them are open source, for anyone else
who wants to build on Mon.
