---
title: "AWS AI Services Can Help You Improve Your Foreign Language"
author: ""
type: ""
date: 2021-04-26T12:12:00+03:00
subtitle: ""
image: ""
tags: []
url: /post/2021/04/26/aws-foreign-language
private: true
---
AWS provides several Artificial Intelligence (AI) services. With AI services, you could implement some useful AI things: image and video analysis, document analysis, text to speech or speech to text translation, and so on. However, these AWS services can be used not only for enterprise applications but also for your self-development applications.

<!--more-->

We are interested in three AI services:
* [Polly](https://aws.amazon.com/polly) — text to speech service. It generates audio by text.
* [Transcribe](https://aws.amazon.com/transcribe) — speech to text service. It recognizes speech to text.
* [Translate](https://aws.amazon.com/translate) — translate service. It translates a text from one language to another one.

## Skills
Applying these services, we will build an application to improve our foreign language skills.

Let’s map AWS AI services to language skills:
* Audition skills — `Polly` allows you to generate audio by text that will help you hear how to pronounce unknown words and phrases.
* Pronunciation skills — `Transcribe` allows you to recognize a speech to a text that will help you train to pronounce new words and phrases.
* Vocabulary volume — `Translate` allows you to translate words, phrases, or whole text from your language to a foreign language or vice versa that will help you understand unknown words and phrases.

The services don't cover all skills, but we develop some of them this way.

***Note***: AWS AI services have different sets of supported languages so there is a chance that one of them hasn't support for your language. Also, the services support a limited number of languages.

## Demo application
I wrote a demo application to show you a proof of concept. The demo app locates in [my GitHub repository](https://github.com/jaitl/aws-lambda-telegram-lang-demo).

I used Telegram Bot API as a UI for the application. However, you could choose any UI you want, for example, CLI, Web, Native OS forms, and so on.

Telegram provides several message types suitable for us:
* Voice message — we could download this message as an `OGG OPUS` file. AWS Transcribe service accepts `OGG OPUS` format as input for speech recognition. So we will be able to send a Telegram voice message right to Transcribe service with no converting.
* Audio message — we could send an MP3 file as an audio message. AWS Polly service returns an MP3 file so we will be able to send the MP3 file to Telegram with no converting too.

I take advantage of these two message types for implementing two main features of the demo application:
* The user sends a voice message then he receives a text of the message as a response.
* The user sends a text message then he receives an audio message and translation from the text as a response.

### Voice message recognition
As I said to recognize a voice message, we call AWS Transcribe service. Transcribe service supports streaming, so we stream a Telegram voice message to this one service. Transcribe Streaming requires three utility classes [AudioStreamPublisher](https://github.com/jaitl/aws-lambda-telegram-lang-demo/blob/main/src/main/kotlin/com/github/jaitl/aws/telegram/english/aws/steamming/AudioStreamPublisher.kt), [SubscriptionImpl](https://github.com/jaitl/aws-lambda-telegram-lang-demo/blob/main/src/main/kotlin/com/github/jaitl/aws/telegram/english/aws/steamming/SubscriptionImpl.kt), and [StreamResponseHandler](https://github.com/jaitl/aws-lambda-telegram-lang-demo/blob/main/src/main/kotlin/com/github/jaitl/aws/telegram/english/aws/steamming/StreamResponseHandler.kt). These files don’t contain business logic so won’t show their code here.

The `transcribe` function that accepts binary stream as an input, creates a request to Transcribe Streaming and returns a recognized text.

```kotlin
fun transcribe(inputAudio: InputStream): String {
    val request = StartStreamTranscriptionRequest.builder()
        .languageCode(LanguageCode.EN_US.toString())
        .mediaEncoding(MediaEncoding.OGG_OPUS)
        .mediaSampleRateHertz(48000)
        .build()

    val blockingQueue: BlockingQueue<String> = LinkedBlockingDeque(100)

    val result = transcribeStreamingClient.startStreamTranscription(
        request,
        AudioStreamPublisher(inputAudio),
        StreamResponseHandler.createResponseHandler(blockingQueue)
    )

    result.get()

    return blockingQueue.last()
}
```

The `handleCommand` function receives a Telegram voice message file as a stream, calls ours `transcribe` function with the stream, receives the recognized text of the voice message, and returns the text to the user.

```kotlin
fun handleCommand(chatId: Long, fileId: String) {
    val fileSteam = telegramBot.getFileStream(fileId)
    val msg = aws.transcribe(fileSteam)
    telegramBot.sendMessage(chatId, "You said: $msg")
}
```

We don’t download the Telegram voice message file to our file system because we get a stream from the file and send the stream straight to Transcribe service.

### Text translation
To translate a text from one language to another one we call AWS Translate service. The `translate` function accepts a text then sends the text to the service, and returns a translated text.

```kotlin
fun translate(text: String): String {
    val textRequest = TranslateTextRequest.builder()
        .sourceLanguageCode("en")
        .targetLanguageCode("ru")
        .text(text)
        .build()

    return translateClient.translateText(textRequest).translatedText()
}
```

### Audio message generation
To generate an audio file from a text we call AWS Polly service. The `synthesizeSpeech` function accepts the text as an input, calls the service and, returns a byte array with a generated MP3 file.

```kotlin
fun synthesizeSpeech(text: String): ByteArray {
    val synthReq: SynthesizeSpeechRequest = SynthesizeSpeechRequest.builder()
        .text(text)
        .voiceId(voice.id())
        .outputFormat(OutputFormat.MP3)
        .build()

    val synthRes = pollyClient.synthesizeSpeech(synthReq)
    val fileByteArray = IoUtils.toByteArray(synthRes)

    synthRes.close()

    return fileByteArray
}
```

The `handleCommand` function accepts a text message from Telegram, calls our `translate` function to translate the text, and calls our `synthesizeSpeech` function to generate an MP3 file then sends results back to the user.

```kotlin
fun handleCommand(chatId: Long, text: String) {
    val translatedText = aws.translate(text)
    val fileByteArray = aws.synthesizeSpeech(text)
    val fileName = UUID.randomUUID().toString().substring(0, 6)
    telegramBot.sendMessage(chatId, "Translation: $translatedText")
    telegramBot.sendAudio(chatId, fileByteArray, title = "$fileName.mp3")
}
```

In this case, we also don’t use our file system because we save the MP3 file to the RAM and we send the file from the RAM straight to Telegram.

I showed only significant parts of my demo project code. Another code is located in [my GitHub repository](https://github.com/jaitl/aws-lambda-telegram-lang-demo).

## Conclusion
Having Polly, Transcribe, and Translate AWS AI services, we implemented a tool that helps us improve some of our foreign language skills. However, the concept presented in this article is only a core for further development.

Based on this core you could implement additional features like that:
* saving original/translated text and audio as [flashcards](https://en.wikipedia.org/wiki/Flashcard)
* exercise with a saved set of flashcards
* gathering statistics about exercise results
* implementing [spaced repetition](https://en.wikipedia.org/wiki/Spaced_repetition) based on the gathered statistic

And so on. What you will add to this core depends only on your requirements and imagination.

Thanks for reading. I hope this was helpful. If you have any questions, feel free to leave a response.

## Resources
* [Demo app repository](https://github.com/jaitl/aws-lambda-telegram-lang-demo)
