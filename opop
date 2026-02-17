package com.example.radarapp // نام پکیج خودت رو بذار

import android.Manifest
import android.bluetooth.*
import android.bluetooth.le.ScanCallback
import android.bluetooth.le.ScanResult
import android.content.pm.PackageManager
import android.os.Bundle
import android.util.Log
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.drawscope.Stroke
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import kotlinx.coroutines.*
import java.io.IOException
import java.io.InputStream
import java.util.*

class MainActivity : ComponentActivity() {

    // --- مدیریت بلوتوث و وضعیت اپلیکیشن ---
    private val bluetoothAdapter: BluetoothAdapter? by lazy {
        val bluetoothManager = getSystemService(BluetoothManager::class.java)
        bluetoothManager?.adapter
    }

    // State مدیریت شده با Compose
    private val bluetoothDevices = mutableStateListOf<BluetoothDevice>()
    private val connectedDevice = mutableState<BluetoothDevice?>(null)
    private val connectionStatus = mutableStateOf("قطع")
    private val radarPoints = mutableStateListOf<Pair<Float, Float>>() // زاویه و فاصله

    // برای ارتباط با سوکت بلوتوث
    private var bluetoothSocket: BluetoothSocket? = null
    private var inputStream: InputStream? = null
    private var isReceivingData by mutableStateOf(false)
    private val receiveJob = Job()
    private val receiveScope = CoroutineScope(Dispatchers.IO + receiveJob)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // درخواست مجوزهای لازم (برای اندروید 6 به بعد)
        requestPermissions()

        setContent {
            RadarAppContent(
                bluetoothDevices = bluetoothDevices,
                connectedDevice = connectedDevice.value,
                connectionStatus = connectionStatus.value,
                radarPoints = radarPoints.toList(),
                onDeviceClick = { device ->
                    connectToDevice(device)
                },
                onDisconnectClick = {
                    disconnectBluetooth()
                },
                onStartScanClick = {
                    scanDevices()
                }
            )
        }
    }

    // --- توابع مدیریت بلوتوث ---

    private fun requestPermissions() {
        val permissions = mutableListOf<String>()
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.BLUETOOTH_SCAN) != PackageManager.PERMISSION_GRANTED) {
            permissions.add(Manifest.permission.BLUETOOTH_SCAN)
        }
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.BLUETOOTH_CONNECT) != PackageManager.PERMISSION_GRANTED) {
            permissions.add(Manifest.permission.BLUETOOTH_CONNECT)
        }
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED) {
            permissions.add(Manifest.permission.ACCESS_FINE_LOCATION)
        }
        if (permissions.isNotEmpty()) {
            ActivityCompat.requestPermissions(this, permissions.toTypedArray(), 1)
        }
    }

    private val leScanCallback = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, result: ScanResult) {
            super.onScanResult(callbackType, result)
            val device = result.device
            // فیلتر برای پیدا کردن دستگاه‌های بلوتوث کلاسیک (HC-05/06)
            if (device.name?.contains("HC", ignoreCase = true) == true) {
                if (!bluetoothDevices.contains(device)) {
                    bluetoothDevices.add(device)
                }
            }
        }
    }

    private fun scanDevices() {
        bluetoothDevices.clear()
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.BLUETOOTH_SCAN) == PackageManager.PERMISSION_GRANTED) {
            bluetoothAdapter?.bluetoothLeScanner?.startScan(leScanCallback)
            connectionStatus.value = "در حال اسکن..."
        }
    }

    private fun connectToDevice(device: BluetoothDevice) {
        if (ActivityCompat.checkSelfPermission(this, Manifest.permission.BLUETOOTH_CONNECT) != PackageManager.PERMISSION_GRANTED) {
            return
        }
        connectionStatus.value = "در حال اتصال..."
        // ایجاد سوکت با UUID استاندارد سریال پورت بلوتوث
        val uuid = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB")
        try {
            bluetoothSocket = device.createRfcommSocketToServiceRecord(uuid)
            bluetoothSocket?.connect()
            inputStream = bluetoothSocket?.inputStream
            connectedDevice.value = device
            connectionStatus.value = "متصل به ${device.name}"
            startReceivingData()
        } catch (e: IOException) {
            e.printStackTrace()
            connectionStatus.value = "خطا در اتصال"
        }
    }

    private fun disconnectBluetooth() {
        receiveJob.cancel()
        try {
            bluetoothSocket?.close()
        } catch (e: IOException) {
            e.printStackTrace()
        }
        bluetoothSocket = null
        inputStream = null
        connectedDevice.value = null
        connectionStatus.value = "قطع"
        radarPoints.clear()
    }

    // --- دریافت و پردازش داده از آردوینو ---
    private fun startReceivingData() {
        receiveScope.launch {
            isReceivingData = true
            val buffer = ByteArray(1024)
            while (isReceivingData) {
                try {
                    if (inputStream == null) break
                    val bytesAvailable = inputStream?.available()
                    if (bytesAvailable != null && bytesAvailable > 0) {
                        val bytesRead = inputStream?.read(buffer, 0, buffer.size)
                        if (bytesRead != null && bytesRead > 0) {
                            val data = String(buffer, 0, bytesRead, Charsets.UTF_8)
                            Log.d("Bluetooth", "Received: $data")
                            parseAndAddRadarPoint(data)
                        }
                    } else {
                        delay(100)
                    }
                } catch (e: IOException) {
                    e.printStackTrace()
                    isReceivingData = false
                }
            }
        }
    }

    private fun parseAndAddRadarPoint(data: String) {
        // داده دریافتی: "فاصله|زاویه|1|1" (مثلاً "20|45|1|1")
        val parts = data.trim().split("|")
        if (parts.size >= 2) {
            try {
                val distance = parts[0].toFloatOrNull()
                val angle = parts[1].toFloatOrNull()
                if (distance != null && angle != null) {
                    withContext(Dispatchers.Main) {
                        // اضافه کردن نقطه جدید به لیست
                        radarPoints.add(0, Pair(angle, distance)) // اضافه به اول لیست
                        // محدود کردن تعداد نقاط برای جلوگیری از سنگین شدن
                        if (radarPoints.size > 100) {
                            radarPoints.removeAt(radarPoints.lastIndex)
                        }
                    }
                }
            } catch (e: Exception) {
                e.printStackTrace()
            }
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        disconnectBluetooth()
        receiveJob.cancel()
    }
}

// --- بخش رابط کاربری با Jetpack Compose ---
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun RadarAppContent(
    bluetoothDevices: List<BluetoothDevice>,
    connectedDevice: BluetoothDevice?,
    connectionStatus: String,
    radarPoints: List<Pair<Float, Float>>,
    onDeviceClick: (BluetoothDevice) -> Unit,
    onDisconnectClick: () -> Unit,
    onStartScanClick: () -> Unit
) {
    val context = LocalContext.current
    var selectedTab by remember { mutableIntStateOf(0) }
    val tabs = listOf("رادار", "دستگاه‌ها")

    Scaffold(
        topBar = {
            Column {
                TopAppBar(
                    title = { Text("رادار بلوتوث") },
                    colors = TopAppBarDefaults.smallTopAppBarColors(
                        containerColor = MaterialTheme.colorScheme.primaryContainer
                    )
                )
                TabRow(selectedTabIndex = selectedTab) {
                    tabs.forEachIndexed { index, title ->
                        Tab(
                            selected = selectedTab == index,
                            onClick = { selectedTab = index },
                            text = { Text(title) }
                        )
                    }
                }
                // وضعیت اتصال
                Row(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(8.dp),
                    horizontalArrangement = Arrangement.SpaceBetween,
                    verticalAlignment = Alignment.CenterVertically
                ) {
                    Text("وضعیت: $connectionStatus")
                    if (connectedDevice != null) {
                        Button(onClick = onDisconnectClick) {
                            Text("قطع اتصال")
                        }
                    } else {
                        Button(onClick = onStartScanClick) {
                            Text("اسکن دستگاه‌ها")
                        }
                    }
                }
            }
        }
    ) { paddingValues ->
        Box(modifier = Modifier.padding(paddingValues)) {
            when (selectedTab) {
                0 -> RadarScreen(radarPoints = radarPoints)
                1 -> DevicesScreen(devices = bluetoothDevices, onDeviceClick = onDeviceClick)
            }
        }
    }
}

@Composable
fun DevicesScreen(
    devices: List<BluetoothDevice>,
    onDeviceClick: (BluetoothDevice) -> Unit
) {
    LazyColumn {
        items(devices.size) { index ->
            val device = devices[index]
            Card(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(8.dp)
                    .clickable { onDeviceClick(device) }
            ) {
                Column(modifier = Modifier.padding(16.dp)) {
                    Text(text = device.name ?: "بدون نام", style = MaterialTheme.typography.titleMedium)
                    Text(text = device.address, style = MaterialTheme.typography.bodySmall)
                }
            }
        }
    }
}

@Composable
fun RadarScreen(radarPoints: List<Pair<Float, Float>>) {
    Canvas(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        val centerX = size.width / 2
        val centerY = size.height / 2
        val radius = minOf(centerX, centerY) * 0.9f

        // رسم دایره‌های رادار (محدوده‌های فاصله)
        for (i in 1..3) {
            drawCircle(
                color = Color.Green,
                radius = radius * (i / 3f),
                style = Stroke(width = 2f),
                center = Offset(centerX, centerY)
            )
        }

        // رسم خطوط زاویه (شعاع‌ها)
        for (angle in 0 until 360 step 45) {
            val rad = Math.toRadians(angle.toDouble())
            val endX = centerX + radius * cos(rad).toFloat()
            val endY = centerY + radius * sin(rad).toFloat()
            drawLine(
                color = Color.Green,
                start = Offset(centerX, centerY),
                end = Offset(endX, endY),
                strokeWidth = 1f
            )
        }

        // رسم نقاط (اشیاء شناسایی شده)
        radarPoints.forEach { point ->
            val (angleDegrees, distanceCm) = point
            // تبدیل زاویه و فاصله به مختصات روی صفحه
            val angleRad = Math.toRadians(angleDegrees.toDouble())
            // در اینجا فرض می‌کنیم حداکثر فاصله قابل تشخیص 200 سانتی‌متر است
            val maxDistance = 200f
            val distanceRatio = (distanceCm / maxDistance).coerceIn(0f, 1f)
            val pointRadius = radius * distanceRatio

            // توجه: در صفحه‌نمایش، زاویه 0 درجه به سمت راست است. اگر سنسور شما در زاویه 0 رو به جلوست، ممکنه نیاز به تنظیم داشته باشی.
            val x = centerX + pointRadius * sin(angleRad).toFloat()
            val y = centerY - pointRadius * cos(angleRad).toFloat()

            drawCircle(
                color = Color.Red,
                radius = 5f,
                center = Offset(x, y)
            )
        }
    }
}

// برای استفاده از Modifier.clickable در DevicesScreen
import androidx.compose.foundation.clickable
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import kotlin.math.cos
import kotlin.math.sin<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- مجوزهای بلوتوث -->
    <uses-permission android:name="android.permission.BLUETOOTH" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
    <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
    <uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
    <!-- مجوز موقعیت مکانی برای اسکن بلوتوث در اندروید 10+ -->
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

    <application
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.MyApplication"
        tools:targetApi="31">
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:theme="@style/Theme.MyApplication">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>dependencies {
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.7.0")
    implementation("androidx.activity:activity-compose:1.8.0")
    implementation(platform("androidx.compose:compose-bom:2024.02.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-graphics")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.foundation:foundation")
}
