package com.example.pfdsimulator
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.Canvas
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.gestures.detectDragGestures
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.geometry.Size
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.Path
import androidx.compose.ui.graphics.drawscope.Stroke
import androidx.compose.ui.graphics.drawscope.rotate
import androidx.compose.ui.graphics.drawscope.translate
import androidx.compose.ui.input.pointer.pointerInput
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.IntOffset
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import kotlinx.coroutines.delay
import kotlin.math.cos
import kotlin.math.roundToInt
import kotlin.math.sin

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = Color(0xFF0F0F14) // Kokpit Paneli Rengi
                ) {
                    BoeingPfdSimulator()
                }
            }
        }
    }
}

@Composable
fun BoeingPfdSimulator() {
    // Uçuş Verileri
    var pitch by remember { mutableStateOf(0f) }
    var roll by remember { mutableStateOf(0f) }
    var altitude by remember { mutableStateOf(1400f) }
    var speed by remember { mutableStateOf(175f) }
    var heading by remember { mutableStateOf(104f) }

    // Joystick Verileri (-1f ile 1f arası)
    var leftJoyX by remember { mutableStateOf(0f) }
    var leftJoyY by remember { mutableStateOf(0f) }
    var rightJoyX by remember { mutableStateOf(0f) }
    var rightJoyY by remember { mutableStateOf(0f) }

    // Oyun Motoru (60 FPS Fizik Döngüsü)
    LaunchedEffect(Unit) {
        while (true) {
            // Sol Joystick: Lövye (Pitch ve Roll)
            pitch = leftJoyY * -30f // +/- 30 derece pitch
            roll = leftJoyX * 45f   // +/- 45 derece yatış

            // Sağ Joystick: Gaz ve Yönlendirme (Speed ve Heading)
            speed -= rightJoyY * 0.5f // Yukarı ittikçe eksi Y gelir, hızı artırırız
            speed = speed.coerceIn(0f, 400f) // Hız sınırı

            if (speed > 40f) { // Uçak hareket ediyorsa yön dönebilir
                heading += rightJoyX * 1.5f
                if (heading >= 360f) heading -= 360f
                if (heading < 0f) heading += 360f

                // İrtifa değişimi pitch ve hıza bağlı
                altitude += pitch * (speed / 150f) * 0.1f
            }

            delay(16L)
        }
    }

    Column(modifier = Modifier.fillMaxSize()) {

        // ÜST KISIM: BOEING PFD EKRANI
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .weight(1.5f)
                .padding(top = 40.dp, start = 16.dp, end = 16.dp, bottom = 16.dp)
                .background(Color.Black)
        ) {
            // FMA (Flight Mode Annunciator) - Yeşil Yazılar
            Row(
                modifier = Modifier.fillMaxWidth().height(30.dp),
                horizontalArrangement = Arrangement.SpaceEvenly,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text("FMC SPD", color = Color.Green, fontWeight = FontWeight.Bold, fontSize = 14.sp)
                Text("LNAV", color = Color.Green, fontWeight = FontWeight.Bold, fontSize = 14.sp)
                Text("VNAV PTH", color = Color.Green, fontWeight = FontWeight.Bold, fontSize = 14.sp)
            }

            // PFD Ana Göstergeleri
            Row(modifier = Modifier.fillMaxSize()) {

                // SOL: Hız Bandı (Speed Tape)
                Box(modifier = Modifier.width(70.dp).fillMaxHeight().background(Color(0xFF333333))) {
                    SpeedTape(speed)
                }

                // ORTA: Suni Ufuk (Attitude Indicator)
                Box(
                    modifier = Modifier
                        .weight(1f)
                        .fillMaxHeight()
                        .clip(RoundedCornerShape(16.dp)) // Resimdeki gibi köşeleri yuvarlak
                ) {
                    AttitudeCanvas(pitch, roll)
                }

                // SAĞ: İrtifa Bandı (Altitude Tape)
                Box(modifier = Modifier.width(80.dp).fillMaxHeight().background(Color(0xFF333333))) {
                    AltitudeTape(altitude)
                }
            }
        }

        // PFD ALT: Heading Göstergesi
        Box(modifier = Modifier.fillMaxWidth().height(60.dp).background(Color.Black), contentAlignment = Alignment.TopCenter) {
            Text(
                text = heading.roundToInt().toString().padStart(3, '0') + " H",
                color = Color.Magenta,
                fontSize = 24.sp,
                fontWeight = FontWeight.Bold
            )
            // Ufak bir pusula yayı (görsel)
            Canvas(modifier = Modifier.fillMaxSize()) {
                drawArc(color = Color.White, startAngle = 180f, sweepAngle = 180f, useCenter = false,
                    topLeft = Offset(size.width/2 - 100f, 10f), size = Size(200f, 200f), style = Stroke(width = 2f))
            }
        }

        // ALT KISIM: 2 ADET JOYSTICK (KONTROL PANELİ)
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .weight(1f)
                .background(Color(0xFF1E1E1E))
                .padding(16.dp),
            horizontalArrangement = Arrangement.SpaceAround,
            verticalAlignment = Alignment.CenterVertically
        ) {
            // SOL JOYSTICK (Pitch / Roll)
            VirtualJoystick(
                label = "YOKE (Pitch/Roll)",
                color = Color.Red,
                onPositionChange = { x, y -> leftJoyX = x; leftJoyY = y }
            )

            // SAĞ JOYSTICK (Gaz / Heading)
            VirtualJoystick(
                label = "THRUST/YAW",
                color = Color.Blue,
                onPositionChange = { x, y -> rightJoyX = x; rightJoyY = y }
            )
        }
    }
}

// SOL HIZ BANDI ÇİZİMİ
@Composable
fun SpeedTape(speed: Float) {
    Box(modifier = Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        // Arka plan çizgileri (Görsel amaçlı)
        Canvas(modifier = Modifier.fillMaxSize()) {
            val h = size.height
            for (i in 0..10) {
                drawLine(Color.White, Offset(size.width - 15f, i * (h / 10)), Offset(size.width, i * (h / 10)), strokeWidth = 2f)
            }
        }
        // Resimdeki siyah gösterge kutusu
        Box(
            modifier = Modifier
                .fillMaxWidth()
                .height(50.dp)
                .background(Color.Black)
                .border(2.dp, Color.White),
            contentAlignment = Alignment.Center
        ) {
            Text(speed.roundToInt().toString(), color = Color.White, fontSize = 24.sp, fontWeight = FontWeight.Bold)
        }
        // Kutunun yanındaki ok işareti
        Canvas(modifier = Modifier.fillMaxSize()) {
            val path = Path().apply {
                moveTo(size.width, size.height / 2 - 10f)
                lineTo(size.width + 10f, size.height / 2)
                lineTo(size.width, size.height / 2 + 10f)
                close()
            }
            drawPath(path, Color.White)
        }
    }
}

// SAĞ İRTİFA BANDI ÇİZİMİ
@Composable
fun AltitudeTape(altitude: Float) {
    Box(modifier = Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        Canvas(modifier = Modifier.fillMaxSize()) {
            val h = size.height
            for (i in 0..10) {
                drawLine(Color.White, Offset(0f, i * (h / 10)), Offset(15f, i * (h / 10)), strokeWidth = 2f)
            }
        }
        Box(
            modifier = Modifier
                .fillMaxWidth()
                .height(50.dp)
                .background(Color.Black)
                .border(2.dp, Color.White),
            contentAlignment = Alignment.CenterStart
        ) {
            Text(" " + altitude.roundToInt().toString(), color = Color.White, fontSize = 22.sp, fontWeight = FontWeight.Bold)
        }
    }
}

// ORTA SUNİ UFUK (BOEING STİLİ) ÇİZİMİ
@Composable
fun AttitudeIndicator(pitch: Float, roll: Float) {
    // (AttitudeCanvas adıyla aşağıda çağrıldı)
}

@Composable
fun AttitudeCanvas(pitch: Float, roll: Float) {
    Canvas(modifier = Modifier.fillMaxSize()) {
        val w = size.width
        val h = size.height
        val center = Offset(w / 2, h / 2)

        rotate(degrees = -roll, pivot = center) {
            translate(top = pitch * 6f) { // Pitch hassasiyeti
                // Mavi Gökyüzü (Boeing Mavisi)
                drawRect(color = Color(0xFF0072B9), topLeft = Offset(-w, -h*2), size = Size(w * 3, h * 2.5f))
                // Kahverengi Yer
                drawRect(color = Color(0xFF8B4513), topLeft = Offset(-w, h / 2), size = Size(w * 3, h * 2.5f))
                // Beyaz Ufuk Çizgisi
                drawLine(color = Color.White, start = Offset(-w, h / 2), end = Offset(w * 2, h / 2), strokeWidth = 4f)

                // Pitch Ladder (Derece Çizgileri)
                for (i in -3..3) {
                    if (i != 0) {
                        val yOffset = h / 2 - (i * 60f)
                        drawLine(Color.White, Offset(center.x - 40f, yOffset), Offset(center.x + 40f, yOffset), strokeWidth = 3f)
                        // Çizgi uçlarındaki ufak kulakçıklar
                        drawLine(Color.White, Offset(center.x - 40f, yOffset), Offset(center.x - 40f, yOffset + (if(i>0) 10f else -10f)), strokeWidth = 3f)
                        drawLine(Color.White, Offset(center.x + 40f, yOffset), Offset(center.x + 40f, yOffset + (if(i>0) 10f else -10f)), strokeWidth = 3f)
                    }
                }
            }
        }

        // Roll Açısı Yayı (En üstteki beyaz yarım çember ve ok)
        drawArc(color = Color.White, startAngle = 210f, sweepAngle = 120f, useCenter = false,
            topLeft = Offset(center.x - 120f, center.y - 120f), size = Size(240f, 240f), style = Stroke(width = 3f))

        val trianglePath = Path().apply {
            moveTo(center.x, center.y - 120f)
            lineTo(center.x - 15f, center.y - 100f)
            lineTo(center.x + 15f, center.y - 100f)
            close()
        }
        drawPath(trianglePath, Color.White)

        // Merkezdeki Uçak İkonu (Boeing tarzı ortası kareli siyah/beyaz çizgiler)
        val aircraftPath = Path().apply {
            moveTo(center.x - 60f, center.y)
            lineTo(center.x - 20f, center.y)
            lineTo(center.x - 20f, center.y + 10f)

            moveTo(center.x + 60f, center.y)
            lineTo(center.x + 20f, center.y)
            lineTo(center.x + 20f, center.y + 10f)
        }
        drawPath(aircraftPath, Color.Black, style = Stroke(width = 12f))
        drawPath(aircraftPath, Color.White, style = Stroke(width = 6f))

        // Merkez Kare
        drawRect(color = Color.Black, topLeft = Offset(center.x - 5f, center.y - 5f), size = Size(10f, 10f))
        drawRect(color = Color.White, topLeft = Offset(center.x - 3f, center.y - 3f), size = Size(6f, 6f), style = Stroke(width = 2f))
    }
}

// SANA ÖZEL İKİLİ JOYSTICK KOMPONENTİ
@Composable
fun VirtualJoystick(label: String, color: Color, onPositionChange: (Float, Float) -> Unit) {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        Text(label, color = Color.White, fontSize = 12.sp, fontWeight = FontWeight.Bold)
        Spacer(modifier = Modifier.height(8.dp))

        Box(
            modifier = Modifier
                .size(140.dp)
                .background(Color(0xFF333333), CircleShape)
                .border(2.dp, Color.Gray, CircleShape),
            contentAlignment = Alignment.Center
        ) {
            var dragOffset by remember { mutableStateOf(Offset.Zero) }
            val maxRadius = 140f / 2f

            Box(
                modifier = Modifier
                    .offset { IntOffset(dragOffset.x.roundToInt(), dragOffset.y.roundToInt()) }
                    .size(50.dp)
                    .background(color, CircleShape)
                    .border(2.dp, Color.White, CircleShape)
                    .pointerInput(Unit) {
                        detectDragGestures(
                            onDragEnd = {
                                dragOffset = Offset.Zero
                                onPositionChange(0f, 0f) // Parmağı çekince sıfırla
                            }
                        ) { change, dragAmount ->
                            change.consume()
                            val newOffset = dragOffset + dragAmount
                            // Joystick dışarı çıkmasın diye sınırla
                            if (newOffset.getDistance() <= maxRadius) {
                                dragOffset = newOffset
                            } else {
                                dragOffset = newOffset / newOffset.getDistance() * maxRadius
                            }
                            // -1.0 ile 1.0 arasında değer döndür
                            onPositionChange(dragOffset.x / maxRadius, dragOffset.y / maxRadius)
                        }
                    }
            )
        }
    }s
}
