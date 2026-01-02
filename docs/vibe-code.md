Code StructureCreate a new Android Studio project (min SDK 24) and add these files. Assumes basic setup; add Retrofit/Gson to build.gradle as shown.build.gradle.kts (Module)

plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.redacted.iptrace"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.redacted.iptrace"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"
    }

    buildFeatures {
        compose = true
    }
    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.1"
    }
}

dependencies {
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.7.0")
    implementation("androidx.activity:activity-compose:1.8.2")
    implementation(platform("androidx.compose:compose-bom:2023.08.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-graphics")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")

    // Retrofit + Gson
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")
}

AndroidManifest.xml

<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.IPTrace">
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:label="@string/app_name"
            android:theme="@style/Theme.IPTrace">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>

MainActivity.kt

package com.redacted.iptrace

import android.content.Intent
import android.net.Uri
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import kotlinx.coroutines.launch
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import retrofit2.http.GET
import retrofit2.http.Path

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                Surface(modifier = Modifier.fillMaxSize()) {
                    IPTraceScreen()
                }
            }
        }
    }
}

data class IpInfo(
    val query: String,
    val country: String,
    val regionName: String,
    val city: String,
    val zip: String,
    val lat: Double,
    val lon: Double,
    val timezone: String,
    val isp: String,
    val org: String
)

interface IpApi {
    @GET("json/{ip}")
    suspend fun getIpInfo(@Path("ip") ip: String): IpInfo

    @GET("json/")
    suspend fun getMyIpInfo(): IpInfo
}

val retrofit = Retrofit.Builder()
    .baseUrl("http://ip-api.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build()

val api = retrofit.create(IpApi::class.java)

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun IPTraceScreen() {
    var ip by remember { mutableStateOf("") }
    var info by remember { mutableStateOf<IpInfo?>(null) }
    var error by remember { mutableStateOf("") }
    var loading by remember { mutableStateOf(false) }
    val coroutineScope = rememberCoroutineScope()
    val context = LocalContext.current

    LaunchedEffect(Unit) {
        // Auto-trace on launch
        loading = true
        try {
            info = api.getMyIpInfo()
        } catch (e: Exception) {
            error = "Failed to fetch your IP info: ${e.message}"
        } finally {
            loading = false
        }
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text("IP Tracer", style = MaterialTheme.typography.headlineMedium)
        Spacer(Modifier.height(16.dp))

        TextField(
            value = ip,
            onValueChange = { ip = it },
            label = { Text("Enter IP (blank for yours)") },
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(Modifier.height(16.dp))

        Button(onClick = {
            coroutineScope.launch {
                loading = true
                error = ""
                try {
                    info = if (ip.isBlank()) api.getMyIpInfo() else api.getIpInfo(ip)
                } catch (e: Exception) {
                    error = "Error: ${e.message}"
                } finally {
                    loading = false
                }
            }
        }) {
            Text("Trace")
        }

        Spacer(Modifier.height(16.dp))

        if (loading) {
            CircularProgressIndicator()
        } else if (error.isNotBlank()) {
            Text(error, color = MaterialTheme.colorScheme.error)
        } else info?.let { data ->
            Card(modifier = Modifier.fillMaxWidth()) {
                Column(modifier = Modifier.padding(16.dp)) {
                    Text("IP: ${data.query}")
                    Text("Country: ${data.country}")
                    Text("Region: ${data.regionName}")
                    Text("City: ${data.city}")
                    Text("Zip: ${data.zip}")
                    Text("Timezone: ${data.timezone}")
                    Text("ISP: ${data.isp}")
                    Text("Org: ${data.org}")

                    Spacer(Modifier.height(8.dp))

                    Button(onClick = {
                        val uri = Uri.parse("geo:${data.lat},${data.lon}")
                        val intent = Intent(Intent.ACTION_VIEW, uri)
                        context.startActivity(intent)
                    }) {
                        Text("View on Map")
                    }
                }
            }
        }
    }
}
