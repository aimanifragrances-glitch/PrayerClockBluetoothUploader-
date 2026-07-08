# Prayer Clock Bluetooth Uploader

HyperTerminal style Android TXT file uploader designed to send configuration or prayer timing files directly to a microcontroller via an HC-05 Bluetooth module.

## Features
- **HC-05 Bluetooth Support:** Connects wirelessly using standard SPP (Serial Port Profile).
- **TXT File Transfer:** Easily browse and select text files from your Android device.
- **9600 Baud Configuration:** Matches the default baud rate of most HC-05 setups.
- **8N1 Serial Communication:** Standard framing used by HyperTerminal.
- **Character Delay:** Adds custom delay (in milliseconds) between characters to prevent buffer overflow on the clock's microcontroller.
- **Line Delay:** Adds custom delay (in milliseconds) after each line transmission.

---

## Android Project Implementation Guide

### 1. Permissions (`AndroidManifest.xml`)
Add the following permissions inside the `<manifest>` tag but outside the `<application>` block:

```xml
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="[http://schemas.android.com/apk/res/android](http://schemas.android.com/apk/res/android)"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Select HC-05 Device:"
        android:textSize="16sp" />

    <Spinner
        android:id="@+id/deviceSpinner"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="8dp" />

    <EditText
        android:id="@+id/etCharDelay"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:hint="Character Delay (ms)"
        android:inputType="number"
        android:text="5" />

    <EditText
        android:id="@+id/etLineDelay"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="8dp"
        android:hint="Line Delay (ms)"
        android:inputType="number"
        android:text="100" />

    <Button
        android:id="@+id/btnSelectFile"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="24dp"
        android:text="Select TXT File" />

    <TextView
        android:id="@+id/tvFilePath"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="8dp"
        android:text="No file selected" />

    <Button
        android:id="@+id/btnUpload"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="24dp"
        android:text="Upload to Prayer Clock" />
</LinearLayout>
import android.bluetooth.BluetoothAdapter;
import android.bluetooth.BluetoothDevice;
import android.bluetooth.BluetoothSocket;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.widget.ArrayAdapter;
import android.widget.Button;
import android.widget.EditText;
import android.widget.Spinner;
import android.widget.TextView;
import android.widget.Toast;
import androidx.appcompat.app.AppCompatActivity;
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.util.ArrayList;
import java.util.Set;
import java.util.UUID;

public class MainActivity extends AppCompatActivity {

    private Spinner deviceSpinner;
    private EditText etCharDelay, etLineDelay;
    private TextView tvFilePath;
    private Uri selectedFileUri = null;
    private ArrayList<BluetoothDevice> pairedDevicesList = new ArrayList<>();
    private static final UUID MY_UUID = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB");

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        deviceSpinner = findViewById(R.id.deviceSpinner);
        etCharDelay = findViewById(R.id.etCharDelay);
        etLineDelay = findViewById(R.id.etLineDelay);
        tvFilePath = findViewById(R.id.tvFilePath);
        Button btnSelectFile = findViewById(R.id.btnSelectFile);
        Button btnUpload = findViewById(R.id.btnUpload);

        loadPairedDevices();

        btnSelectFile.setOnClickListener(v -> {
            Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
            intent.setType("text/plain");
            startActivityForResult(intent, 101);
        });

        btnUpload.setOnClickListener(v -> {
            if (selectedFileUri == null) {
                Toast.makeText(this, "Please select a TXT file first!", Toast.LENGTH_SHORT).show();
                return;
            }
            if (deviceSpinner.getSelectedItem() == null) {
                Toast.makeText(this, "Please select a Bluetooth device!", Toast.LENGTH_SHORT).show();
                return;
            }
            
            BluetoothDevice device = pairedDevicesList.get(deviceSpinner.getSelectedItemPosition());
            int charDelay = Integer.parseInt(etCharDelay.getText().toString());
            int lineDelay = Integer.parseInt(etLineDelay.getText().toString());

            new Thread(() -> startUpload(device, charDelay, lineDelay)).start();
        });
    }

    private void loadPairedDevices() {
        BluetoothAdapter bluetoothAdapter = BluetoothAdapter.getDefaultAdapter();
        if (bluetoothAdapter == null) return;

        Set<BluetoothDevice> pairedDevices = bluetoothAdapter.getBondedDevices();
        ArrayList<String> deviceNames = new ArrayList<>();

        for (BluetoothDevice device : pairedDevices) {
            pairedDevicesList.add(device);
            deviceNames.add(device.getName() + "\n" + device.getAddress());
        }

        ArrayAdapter<String> adapter = new ArrayAdapter<>(this, android.R.layout.simple_spinner_item, deviceNames);
        adapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
        deviceSpinner.setAdapter(adapter);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == 101 && resultCode == RESULT_OK && data != null) {
            selectedFileUri = data.getData();
            tvFilePath.setText("Selected: " + selectedFileUri.getLastPathSegment());
        }
    }

    private void startUpload(BluetoothDevice device, int charDelay, int lineDelay) {
        try {
            runOnUiThread(() -> Toast.makeText(this, "Connecting...", Toast.LENGTH_SHORT).show());
            BluetoothSocket socket = device.createRfcommSocketToServiceRecord(MY_UUID);
            socket.connect();
            OutputStream out = socket.getOutputStream();

            BufferedReader br = new BufferedReader(new InputStreamReader(getContentResolver().openInputStream(selectedFileUri)));
            String line;

            runOnUiThread(() -> Toast.makeText(this, "Uploading...", Toast.LENGTH_SHORT).show());

            while ((line = br.readLine()) != null) {
                line = line + "\r\n"; // HyperTerminal standard new line
                for (int i = 0; i < line.length(); i++) {
                    out.write((int) line.charAt(i));
                    out.flush();
                    if (charDelay > 0) Thread.sleep(charDelay);
                }
                if (lineDelay > 0) Thread.sleep(lineDelay);
            }

            br.close();
            out.close();
            socket.close();

            runOnUiThread(() -> Toast.makeText(this, "Upload Success!", Toast.LENGTH_LONG).show());
        } catch (Exception e) {
            e.printStackTrace();
            runOnUiThread(() -> Toast.makeText(this, "Error: " + e.getMessage(), Toast.LENGTH_LONG).show());
        }
    }
}
