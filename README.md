# I Only Wanted a RAG for Mon, and Ended Up Building an OCR

_Mon shares its script with Burmese, so the first models I tried read it as Burmese. This is how a small RAG turned into an OCR, and what I learned about text on the way._

![Article cover - A page of text lines beside a grid of stored characters, with one row out of line](./assets/i-only-wanted-a-rag-for-mon.avif)

I wanted a small RAG for Mon, my native language. The idea was simple: collect some Mon text, make
it searchable, and ask it questions. I thought it would take a weekend.

Mon is an old language, spoken across Southeast Asia in what is now Myanmar and Thailand. Its
earliest inscriptions are about 1,400 years old. The [last Mon
kingdom](https://en.wikipedia.org/wiki/Mon_kingdoms) fell in 1757, and the language has lost ground
since. UNESCO's 2010 atlas listed it as vulnerable.

Mon also gave Burmese its script. In the 11th century the Burmese king Anawrahta conquered the Mon
city of Thaton, and the Burmese adapted the Mon script to write their own language. The two
languages aren't related, but the scripts are close relatives. Mon still has letters that Burmese
doesn't, so an OCR model trained only on Burmese has no output for them.

It took a lot longer than a weekend. The first models I tried read Mon as Burmese. I couldn't find a
tokenizer, a language detector or a dataset to start from. So I decided to make them, the NLP tools
first. Language models would have to wait. Every one of those needed text.

## Finding the text

I took whatever I could reach: Mon Wikipedia, a news site, posts from Facebook and Telegram, and a
dictionary database. I put it all in one public collection,
[MonCorpusCollection](https://github.com/MonDevHub/MonCorpusCollection). It came to about 47 million
characters, mostly from Wikipedia and the news site. That's too little to train a language model
from scratch.

The most promising text was somewhere else: in books, a pile of scanned ones. But a PDF of a book
doesn't hand you text. Some of them were only page images. A person can read those, but a search
index can't.

## Reading the pages, and why it had to run on a phone

The answer was OCR. Before I built anything, I looked into the tools that already existed: Kraken,
Tesseract, TrOCR and PaddleOCR. I set up Kraken and a TrOCR fine-tuning script in September 2025,
next to a small recogniser of my own. In the end I built my own for Mon. I haven't run a
head-to-head comparison with them on Mon pages yet, so I won't claim how it ranks. That's on my
list.

What shaped the design was who I expected to use it. Most people who'd want to read these books are
on phones, and many are in rural areas with a poor connection. An OCR service that needs an upload
doesn't help someone who can't send a whole book. So I built the mobile model first, small enough to
run on the device and work offline. A bigger server model could come later.

## Three ways to write the same letters

Later I went back to the books that did have a text layer, to see what was in them. I checked
thirteen, and one of them extracts like this:

`k[.j}tY`

On screen, that page shows Mon. In the file it's plain ASCII. The book uses an old font that draws
Mon letters on the slots meant for English ones. The letters only appear when that font is there to
draw them.

Myanmar script has been written down in at least three ways. Unicode gives every letter one agreed
number. Zawgyi was the common way to type Burmese for years. It uses the same numbers but draws some
of them as different letters, so the text looks right in one font and wrong in another. And the old
legacy fonts, like this one, ignore those numbers altogether. Unicode is recent in Myanmar
publishing, and for Mon it was rare. Of the thirteen, ten were legacy font-encoded ASCII and three
were Zawgyi. None was Unicode.

I looked for a converter. Rabbit, the standard Zawgyi converter, is written for Burmese, and as far
as I could see its rules don't name any other language. I couldn't find a reviewed one for Mon. That
matters. The Mon-only letters are rare on a page, but about 60% of the Mon lines I counted use at
least one of the eleven that Burmese doesn't have.

The OCR doesn't care about any of this. The model reads pixels, so it never sees the encoding. A
legacy-font book like that one is among the cleanest results in my test samples. The text layers
were no use to me, and the page images were fine.

## How the model got here

Before the first serious model I had a plain CNN-based recogniser. The serious one used ResNet-18,
which read lines at 64 pixels high. Mon puts vowel signs and medials, the small marks around a
letter, above and below the line. At that height they were hard to resolve. The next one switched to
MobileNetV3 with a taller input. That model needed more capacity for complex diacritic combinations,
and it had no attention.

I also built a first server design, a Swin transformer with an autoregressive decoder. I archived it
before it ever finished training. The design was heavy and a different kind of model to maintain. It
also worked against the whole point: a model that runs on a phone.

The current one, v3.5, is MobileNetV3-Large with two BiLSTM layers, a small attention block and a
CTC head. It has about 11.5 million parameters, and the exported ONNX file is about 46 MB.

The hardest bug wasn't in any of them. I trained on synthetic lines: render text in a font, use that
text as the label. One font, UniMon, turned out to be Zawgyi-encoded. The codepoints were right and
several of the letters drawn were wrong. My validation lines, the ones I held back to score the
model while it trained, used only that font. For a while the scores marked the model wrong for
reading the image correctly.

It was the same kind of problem as those book text layers, and it had been sitting in my own
validation data.

I stopped trusting fonts by name. A font now has to pass a test before it's used. It must draw the
right letter for each codepoint, and it must have the Mon letters at all. The data generator and the
audit run the same test.

## Where it is now

You can use the model three ways. A command-line tool reads whole PDFs and folders of images in
batch. The [web app](https://ocr.mondevhub.com) runs it in the browser. And Android and iOS versions
run it on the device. You have to build those from source for now, since they aren't in the app
stores.

It reads Mon from PDFs whatever their encoding, and from screenshots. In my own use it has also read
photos taken on my phone, and posters. In three published samples, a Zawgyi PDF, a legacy-font PDF
and a typeset screenshot, none of the 563 lines came out as garbage. Those samples were picked from
a screening, so they show it at its best. Across a wider pile of books, about 9% of lines come out
garbled.

On 150 held-out lines, in a typeface the model never saw, about one character in a hundred comes out
wrong. The character error rate is 0.0100, with a 95% interval of 0.0056 to 0.0147. Those lines are
synthetic, and I haven't put a number on photographs yet. The [model
card](https://huggingface.co/janakhpon/monocr) has the details.

The command-line tool is also what brought me back to where I started. I've been using it to read
books from my archive, a page at a time, and the text goes into the same collection, in a folder of
its own. There are 16 books so far. It's machine OCR and nobody has proofread it. Each book has a
record of its source and of which pages were kept. Pages that come out empty or garbled get dropped,
so what's left is text the model produced something sensible for.

## What's next

The corpus grows a book at a time. Next I want a reviewed set of real Mon pages, so I can measure
accuracy where it matters and compare the model with Kraken, Tesseract and the others properly. I
also want a bigger model for hard pages, and its training pipeline is built. Seven of the nine
sources I collected from don't have an established licence yet, so I'm careful about what I say can
be reused.

Looking back, most of the work wasn't training a model. It was finding out what the text really was
before I trusted it: what a font drew, what an encoding meant and what a score was measuring.

The RAG is still unfinished. But books that were only page images last September are text now, and
the tools that read them are open source.
