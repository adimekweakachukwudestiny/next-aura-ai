# next-aura-ai
Next Aura AI - Personal Android Voice Assistant

package com.nextaura.ai

import android.Manifest
import android.app.Activity
import android.content.Intent
import android.content.pm.PackageManager
import android.os.Bundle
import android.speech.RecognizerIntent
import android.speech.SpeechRecognizer
import android.speech.RecognitionListener
import android.speech.tts.TextToSpeech
import android.view.Gravity
import android.widget.*
import android.graphics.Color
import java.text.SimpleDateFormat
import java.util.*

class MainActivity : Activity(), TextToSpeech.OnInitListener {

    private var speechRecognizer: SpeechRecognizer? = null
    private var textToSpeech: TextToSpeech? = null
    private lateinit var status: TextView
    private lateinit var result: TextView

    private val audioPermission = Manifest.permission.RECORD_AUDIO
    private val permissionCode = 100

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        textToSpeech = TextToSpeech(this, this)

        createScreen()

        if (checkSelfPermission(audioPermission) !=
            PackageManager.PERMISSION_GRANTED) {
            requestPermissions(arrayOf(audioPermission), permissionCode)
        } else {
            setupSpeechRecognition()
        }
    }

    private fun createScreen() {
        val layout = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            gravity = Gravity.CENTER
            setPadding(28, 40, 28, 40)
            setBackgroundColor(Color.rgb(13, 17, 42))
        }

        val title = TextView(this).apply {
            text = "NEXT AURA AI"
            textSize = 30f
            setTextColor(Color.WHITE)
            gravity = Gravity.CENTER
        }

        val subtitle = TextView(this).apply {
            text = "Your personal voice assistant"
            textSize = 16f
            setTextColor(Color.LTGRAY)
            gravity = Gravity.CENTER
        }

        status = TextView(this).apply {
            text = "Ready to assist you"
            textSize = 17f
            setTextColor(Color.CYAN)
            gravity = Gravity.CENTER
        }

        val microphone = Button(this).apply {
            text = "🎙 TAP TO SPEAK"
            textSize = 20f
            setOnClickListener { startListening() }
        }

        result = TextView(this).apply {
            text = "Your conversation will appear here."
            textSize = 17f
            setTextColor(Color.WHITE)
            gravity = Gravity.CENTER
        }

        val stopButton = Button(this).apply {
            text = "STOP SPEAKING"
            setOnClickListener {
                textToSpeech?.stop()
                status.text = "Stopped"
            }
        }

        layout.addView(title)
        layout.addView(subtitle)
        layout.addView(status)
        layout.addView(microphone)
        layout.addView(result)
        layout.addView(stopButton)

        setContentView(layout)
    }

    private fun setupSpeechRecognition() {
        if (!SpeechRecognizer.isRecognitionAvailable(this)) {
            status.text = "Speech recognition is unavailable"
            return
        }

        speechRecognizer = SpeechRecognizer.createSpeechRecognizer(this)

        speechRecognizer?.setRecognitionListener(object : RecognitionListener {
            override fun onReadyForSpeech(params: Bundle?) {
                status.text = "Listening..."
            }

            override fun onBeginningOfSpeech() {}
            override fun onRmsChanged(rmsdB: Float) {}
            override fun onBufferReceived(buffer: ByteArray?) {}

            override fun onEndOfSpeech() {
                status.text = "Thinking..."
            }

            override fun onError(error: Int) {
                status.text = "Couldn't hear you. Tap to try again."
            }

            override fun onResults(results: Bundle?) {
                val words = results?.getStringArrayList(
                    SpeechRecognizer.RESULTS_RECOGNITION
                )

                val spokenText = words?.firstOrNull() ?: ""
                result.text = "You said: $spokenText"

                respondTo(spokenText)
            }

            override fun onPartialResults(partialResults: Bundle?) {}
            override fun onEvent(eventType: Int, params: Bundle?) {}
        })
    }

    private fun startListening() {
        if (checkSelfPermission(audioPermission) !=
            PackageManager.PERMISSION_GRANTED) {
            requestPermissions(arrayOf(audioPermission), permissionCode)
            return
        }

        if (speechRecognizer == null) setupSpeechRecognition()

        val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
            putExtra(
                RecognizerIntent.EXTRA_LANGUAGE_MODEL,
                RecognizerIntent.LANGUAGE_MODEL_FREE_FORM
            )
            putExtra(RecognizerIntent.EXTRA_LANGUAGE, Locale.getDefault())
            putExtra(RecognizerIntent.EXTRA_PROMPT, "Speak to Next Aura")
        }

        speechRecognizer?.startListening(intent)
    }

    private fun respondTo(message: String) {
        val input = message.lowercase(Locale.getDefault())

        val answer = when {
            "hello" in input || "hi" in input ->
                "Hello! I am Next Aura. How can I help you?"

            "time" in input ->
                "The time is " +
                SimpleDateFormat("h:mm a", Locale.getDefault()).format(Date())

            "date" in input || "day" in input ->
                "Today is " +
                SimpleDateFormat("EEEE, d MMMM yyyy",
                    Locale.getDefault()).format(Date())

            "gmail" in input -> {
                openApp("com.google.android.gm")
                "Opening Gmail."
            }

            "stop" in input ->
                "Okay. I will stop."

            else ->
                "I heard you say: $message. My full AI connection is not installed yet."
        }

        status.text = "Next Aura is speaking..."
        result.text = "Next Aura: $answer"
        speak(answer)
    }

    private fun openApp(packageName: String) {
        val launchIntent = packageManager
            .getLaunchIntentForPackage(packageName)

        if (launchIntent != null) {
            startActivity(launchIntent)
        }
    }

    private fun speak(message: String) {
        textToSpeech?.speak(
            message,
            TextToSpeech.QUEUE_FLUSH,
            null,
            "NEXT_AURA_REPLY"
        )
    }

    override fun onInit(statusCode: Int) {
        if (statusCode == TextToSpeech.SUCCESS) {
            textToSpeech?.language = Locale.US
            textToSpeech?.setSpeechRate(0.92f)
        }
    }

    override fun onRequestPermissionsResult(
        requestCode: Int,
        permissions: Array<out String>,
        grantResults: IntArray
    ) {
        super.onRequestPermissionsResult(
            requestCode, permissions, grantResults
        )

        if (requestCode == permissionCode &&
            grantResults.isNotEmpty() &&
            grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            setupSpeechRecognition()
            status.text = "Ready to assist you"
        }
    }

    override fun onDestroy() {
        speechRecognizer?.destroy()
        textToSpeech?.stop()
        textToSpeech?.shutdown()
        super.onDestroy()
    }
}
