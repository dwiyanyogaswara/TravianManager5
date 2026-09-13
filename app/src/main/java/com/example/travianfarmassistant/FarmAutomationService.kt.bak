package com.example.travianfarmassistant

import android.annotation.SuppressLint
import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.app.Service
import android.content.Intent
import android.os.Build
import android.os.Handler
import android.os.IBinder
import android.os.Looper
import android.security.keystore.KeyGenParameterSpec
import android.security.keystore.KeyProperties
import android.webkit.CookieManager
import android.webkit.JavascriptInterface
import android.webkit.WebView
import android.webkit.WebViewClient
import android.util.Base64
import javax.crypto.Cipher
import javax.crypto.KeyGenerator
import javax.crypto.spec.GCMParameterSpec
import javax.crypto.spec.SecretKeySpec
import java.security.KeyStore
import org.json.JSONObject
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale
import java.lang.ref.WeakReference
import kotlin.random.Random

class FarmAutomationService : Service() {
    companion object {
        private var instanceRef: WeakReference<FarmAutomationService>? = null
        private var visibleWebViewRef: WeakReference<WebView>? = null

        fun attachVisibleWebView(view: WebView) {
            visibleWebViewRef = WeakReference(view)
        }

        fun detachVisibleWebView(view: WebView) {
            if (visibleWebViewRef?.get() === view) visibleWebViewRef = null
        }

        fun isRunningFromService(): Boolean {
            return instanceRef?.get()?.running == true
        }

        fun forwardPageFinished(url: String) {
            instanceRef?.get()?.handleVisiblePageFinished(url)
        }

        fun forwardLoginResult(result: String) {
            instanceRef?.get()?.handleLoginResultFromVisibleWebView(result)
        }

        fun forwardVillageListResult(result: String) {
            instanceRef?.get()?.handleVillageListResult(result)
        }

        fun onVisibleWebViewDetached() {
            instanceRef?.get()?.onVisibleWebViewDetachedInternal()
        }

        const val ACTION_START = "com.example.travianfarmassistant.START"
        const val ACTION_STOP = "com.example.travianfarmassistant.STOP"
        const val EXTRA_SERVER = "server"
        const val EXTRA_USERNAME = "username"
        const val EXTRA_PASSWORD = "password"
        const val EXTRA_MINUTES_MIN = "minutes_min"
        const val EXTRA_MINUTES_MAX = "minutes_max"
        const val EXTRA_RESOURCE_BUILDER = "resource_builder"
        const val EXTRA_FARM_LIST_ENABLED = "farm_list_enabled"
        const val EXTRA_SELECTED_VILLAGES = "selected_villages"
        const val EXTRA_SELECTED_VILLAGES_JSON = "selected_villages_json"
        const val EXTRA_SELECTION_CONFIGURED = "selection_configured"

        private const val CHANNEL_ID = "farm_automation"
        private const val NOTIFICATION_ID = 2001
        private const val PREFS = "config"
        private const val KEYSTORE = "AndroidKeyStore"
        private const val PASSWORD_KEY_ALIAS = "TravianFarmAssistantPassword"
    }

    private val handler = Handler(Looper.getMainLooper())
    private var webView: WebView? = null
    private var running = false
    private var pendingStartAll = false
    private var loginInProgress = false
    private var reloginRequested = false
    private var loginRetryCount = 0
    private var startAllAttempt = 0
    private var raidCountBeforeStartAll = 0
    private var raidVerificationAttempt = 0
    private var farmListBeforeReady = 0
    private var farmListProgressObserved = false
    private var farmListLastState = ""
    private var farmListStableChecks = 0
    private var fallbackFarmListMode = false
    private var consentAttempt = 0
    private var server = ""
    private var username = ""
    private var password = ""
    private var minMinutes = 1L
    private var maxMinutes = 1L
    private var nextAt = 0L
    private var scheduledRefreshForNextRun = false
    private var countdownCyclePending = false
    private var initialCyclePending = false
    private data class VillageDataRecord(
        val isChecklist: Boolean,
        val namaVillage: String,
        val id: String,
        val linkVillage: String,
        val linkResource: String,
        val resourceId: String,
        val resourceGid: String,
        val minLvl: Int
    )

    private fun loadVillageDataRecordsFromPrefs(): List<VillageDataRecord> {
        debugTrace("ENTER loadVillageDataRecordsFromPrefs")
        val raw = getSharedPreferences(PREFS, MODE_PRIVATE)
            .getString("village_data_json", "[]").orEmpty()
        val array = runCatching { org.json.JSONArray(raw) }.getOrNull() ?: return emptyList()
        val out = mutableListOf<VillageDataRecord>()
        val seen = mutableSetOf<String>()
        for (i in 0 until array.length()) {
            val item = array.optJSONObject(i) ?: continue
            val id = item.optString("Id").trim()
            if (id.isBlank() || !seen.add(id)) continue
            out.add(
                VillageDataRecord(
                    isChecklist = item.optBoolean("IsChecklist", false),
                    namaVillage = item.optString("NamaVillage").trim().ifBlank { "Village $id" },
                    id = id,
                    linkVillage = item.optString("LinkVillage").trim(),
                    linkResource = item.optString("LinkResource").trim(),
                    resourceId = item.optString("ResourceId").trim(),
                    resourceGid = item.optString("ResourceGid").trim(),
                    minLvl = item.optInt("MinLvl", -1)
                )
            )
        }
        return out
    }

    private fun loadBuilderStateFromVillageData(): Boolean {
        debugTrace("ENTER loadBuilderStateFromVillageData")
        val records = loadVillageDataRecordsFromPrefs()
        builderVillages.clear()
        builderVillageLinks.clear()
        builderResourceLinks.clear()
        builderResourceLevels.clear()

        // Record IsChecklist=false tidak pernah masuk ke loop Builder.
        val selected = records.filter { it.isChecklist }
        for (record in selected) {
            builderVillages.add(record.id to record.namaVillage)
            if (record.linkVillage.isNotBlank()) builderVillageLinks[record.id] = record.linkVillage
            if (record.linkResource.isNotBlank()) builderResourceLinks[record.id] = record.linkResource
            if (record.minLvl >= 0) builderResourceLevels[record.id] = record.minLvl
        }

        logEvent(
            "Resource Builder: database village dimuat — total=${records.size}, " +
                "checklist=${selected.size}, resourceLink=${builderResourceLinks.size}"
        )
        selected.forEach { record ->
            logEvent(
                "Resource Builder DB: ${record.namaVillage} [${record.id}] " +
                    "check=${record.isChecklist}; village=${record.linkVillage.ifBlank { "-" }}; " +
                    "resource=${record.linkResource.ifBlank { "-" }}; min=L${record.minLvl}"
            )
        }
        return selected.isNotEmpty()
    }

    private var resourceBuilderEnabled = true
    private var farmListEnabled = true
    private var builderSelectionConfigured = false
    private var selectedBuilderVillageIds = emptySet<String>()
    private var selectedBuilderVillagesJson = "[]"
    private var builderInProgress = false
    private var builderVillages = mutableListOf<Pair<String, String>>()
    private var builderVillageIndex = 0
    private var builderAttempt = 0

    // Target resource disimpan saat scanner UI mencari level terendah.
    // Resource Builder tidak lagi menebak field dari halaman village ketika eksekusi;
    // ia memakai href yang sudah disimpan untuk village tersebut.
    private val builderResourceLinks = linkedMapOf<String, String>()
    private val builderVillageLinks = linkedMapOf<String, String>()
    private val builderResourceLevels = linkedMapOf<String, Int>()
    private var pendingBuilderResourceHref = ""
    private var builderVillageClickInProgress = false
    private var builderDiscoverInFlight = false
    // State machine agar callback onPageFinished tidak menjalankan Builder
    // berulang-ulang pada dorf1.php atau salah mengklik tombol di halaman lain.
    private var builderStage = "IDLE"

    private var pendingUpgradeUrl = ""
    private var pendingUpgradeCosts = longArrayOf(0L, 0L, 0L, 0L)
    private var heroTransferCompleted = false
    private var inventoryUseAttempt = 0
    private var cycleNumber = 0
    private var farmListCycleStartedAt = 0L
    private var resourceBuilderCycleStartedAt = 0L
    private var recoveringService = false
    private var webViewRecoveryInProgress = false
    private var lastAutomationUrl = ""

    private var villageRefreshInProgress = false
    private var villageRefreshCompleted = false
    private var villageRefreshClosed = true
    private var villageRefreshStartedAt = 0L
    private var villageRefreshTimeoutRunnable: Runnable? = null
    private var villageRefreshIndex = 0
    private var villageRefreshRetry = 0
    private var villageRefreshInspectInFlight = false
    private var villageRefreshVillages = mutableListOf<Pair<String, String>>()
    private var farmListCycleComplete = false

    private val cycleWatchdogRunnable: Runnable = Runnable {
        if (!running) return@Runnable
        val now = System.currentTimeMillis()
        persistActiveCycleDuration(now)
        logEvent("WATCHDOG: fase siklus berjalan >5 menit — proses aktif diakhiri agar scheduler tidak stuck")
        pendingStartAll = false
        builderInProgress = false
        loginInProgress = false
        reloginRequested = false
        pendingUpgradeUrl = ""
        pendingUpgradeCosts = longArrayOf(0L, 0L, 0L, 0L)
        heroTransferCompleted = false
        try { automationWebView()?.stopLoading() } catch (_: Exception) {}
        scheduleNextRandomRun()
        updateNotification("Siklus dihentikan oleh watchdog 5 menit")
    }
    /**
     * Refresh Village dijalankan 30 detik setelah countdown dimulai.
     * Refresh adalah pekerjaan persiapan untuk cycle berikutnya dan maksimal 3 menit.
     */
    private val delayedVillageRefreshRunnable: Runnable = object : Runnable {
        override fun run() {
            if (!running) return
            if (villageRefreshInProgress || villageRefreshCompleted) return
            if (pendingStartAll || builderInProgress || loginInProgress || reloginRequested) {
                logEvent("AUTO REFRESH VILLAGE: WebView sedang dipakai; refresh ditunda 10 detik")
                handler.postDelayed(this, 10_000L)
                return
            }
            startAutomaticVillageRefresh()
        }
    }

    private fun scheduleVillageRefreshForNextRun(countdownStartedAt: Long) {
        handler.removeCallbacks(delayedVillageRefreshRunnable)
        if (!running) return
        val refreshAt = countdownStartedAt + 30_000L
        val delay = (refreshAt - System.currentTimeMillis()).coerceAtLeast(0L)
        scheduledRefreshForNextRun = true
        countdownCyclePending = true
        getSharedPreferences(PREFS, MODE_PRIVATE).edit()
            .putLong("countdown_started_at", countdownStartedAt)
            .apply()
        logEvent("AUTO REFRESH VILLAGE: dijadwalkan 30 detik SETELAH Countdown dimulai — ${timeFormat.format(Date(refreshAt))}")
        handler.postDelayed(delayedVillageRefreshRunnable, delay)
    }

    private fun closeAutomaticVillageRefresh(reason: String) {
        if (!villageRefreshInProgress && villageRefreshClosed) return
        villageRefreshTimeoutRunnable?.let { handler.removeCallbacks(it) }
        villageRefreshTimeoutRunnable = null
        villageRefreshInProgress = false
        villageRefreshCompleted = true
        villageRefreshClosed = true
        villageRefreshInspectInFlight = false
        try { automationWebView()?.stopLoading() } catch (_: Exception) {}
        logEvent("AUTO REFRESH VILLAGE: ditutup — $reason")
        updateNotification("Refresh Village selesai")

        if (running && initialCyclePending) {
            initialCyclePending = false
            countdownCyclePending = false
            handler.post { triggerScheduledCycle() }
        } else if (running && nextAt > 0L) {
            // Pastikan callback Next Run tetap terpasang setelah AUTO REFRESH VILLAGE.
            // Refresh memakai Handler yang sama dan pada kondisi tertentu callback
            // countdown dapat hilang dari queue. Jangan menunggu sampai pengguna
            // mematikan/menyalakan bot untuk memulihkan scheduler.
            val remaining = nextAt - System.currentTimeMillis()
            handler.removeCallbacks(nextRunRunnable)
            if (remaining <= 0L) {
                logEvent("Next Run: waktu sudah tiba — lanjut ke cycle")
                handler.post { triggerScheduledCycle() }
            } else {
                handler.postDelayed(nextRunRunnable, remaining)
                logEvent("Next Run: ${timeFormat.format(Date(nextAt))}")
                updateNotification(
                    "Next Run ${timeFormat.format(Date(nextAt))} | dalam ${formatDuration(remaining)}"
                )
            }
        }
    }

    private val timeFormat = SimpleDateFormat("HH:mm:ss", Locale.getDefault())
    private val logTimeFormat = SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.getDefault())
    private val logFileName = "farm_assistant.log"
    private val logMaxAgeMs = 12 * 60 * 60 * 1000L

    override fun onCreate() {
        debugTrace("ENTER onCreate")
        super.onCreate()
        instanceRef = WeakReference(this)
        createNotificationChannel()
        startForeground(NOTIFICATION_ID, buildNotification("Farm Assistant aktif"))
        handler.post { recoverAfterProcessRecreation() }
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        debugTrace("ENTER onStartCommand")
        when (intent?.action) {
            ACTION_STOP -> stopAutomation(startId)
            ACTION_START -> {
                server = normalizeServer(intent.getStringExtra(EXTRA_SERVER).orEmpty())
                username = intent.getStringExtra(EXTRA_USERNAME).orEmpty().trim()
                password = intent.getStringExtra(EXTRA_PASSWORD).orEmpty()
                minMinutes = intent.getLongExtra(EXTRA_MINUTES_MIN, 1L).coerceAtLeast(1L)
                maxMinutes = intent.getLongExtra(EXTRA_MINUTES_MAX, minMinutes).coerceAtLeast(minMinutes)
                resourceBuilderEnabled = intent.getBooleanExtra(EXTRA_RESOURCE_BUILDER, true)
                farmListEnabled = intent.getBooleanExtra(EXTRA_FARM_LIST_ENABLED, true)
                builderSelectionConfigured = intent.getBooleanExtra(
                    EXTRA_SELECTION_CONFIGURED,
                    getSharedPreferences(PREFS, MODE_PRIVATE).getBoolean("resource_builder_selection_configured", false)
                )
                selectedBuilderVillageIds = intent.getStringArrayExtra(EXTRA_SELECTED_VILLAGES)?.toSet()
                    ?: (getSharedPreferences(PREFS, MODE_PRIVATE).getStringSet("resource_builder_selected_villages", emptySet()) ?: emptySet())
                selectedBuilderVillagesJson = intent.getStringExtra(EXTRA_SELECTED_VILLAGES_JSON)
                    ?: getSharedPreferences(PREFS, MODE_PRIVATE).getString("resource_builder_villages_json", "[]").orEmpty()

                // Pastikan service kembali aktif saat Live Bot dinyalakan setelah sebelumnya dimatikan.
                // ACTION_START dapat datang ke instance service baru karena stopAutomation() memanggil stopSelf().
                runCatching {
                    startForeground(NOTIFICATION_ID, buildNotification("Farm Assistant aktif — memulai bot"))
                }
                logEvent("Live Bot ON — ACTION_START diterima; memulai siklus bot sekarang")
                // Pastikan callback/state sisa dari sesi sebelumnya tidak ikut terbawa.
                // Ini penting untuk skenario OFF → ON tanpa menutup aplikasi.
                handler.removeCallbacksAndMessages(null)
                pendingStartAll = false
                builderInProgress = false
                loginInProgress = false
                reloginRequested = false
                villageRefreshInProgress = false
                villageRefreshCompleted = true
                villageRefreshClosed = true
                countdownCyclePending = false
                scheduledRefreshForNextRun = false
                farmListCycleComplete = false
                startAutomation()
            }
            null -> recoverAfterProcessRecreation()
        }
        return START_STICKY
    }

    private fun recoverAfterProcessRecreation() {
        debugTrace("ENTER recoverAfterProcessRecreation")
        val prefs = getSharedPreferences(PREFS, MODE_PRIVATE)
        if (!prefs.getBoolean("service_running", false)) return
        if (recoveringService) return
        recoveringService = true

        val credential = runCatching { CredentialDatabase(this).read() }.getOrNull()
        server = normalizeServer(credential?.server.orEmpty())
        username = credential?.username.orEmpty().trim()
        password = credential?.password.orEmpty()
        minMinutes = prefs.getLong("interval_min_minutes", 1L).coerceAtLeast(1L)
        maxMinutes = prefs.getLong("interval_max_minutes", minMinutes).coerceAtLeast(minMinutes)
        farmListEnabled = prefs.getBoolean("farm_list_enabled", true)
        resourceBuilderEnabled = prefs.getBoolean("resource_builder_enabled", true)
        builderSelectionConfigured = prefs.getBoolean("resource_builder_selection_configured", false)
        selectedBuilderVillageIds = prefs.getStringSet("resource_builder_selected_villages", emptySet()) ?: emptySet()
        selectedBuilderVillagesJson = prefs.getString("resource_builder_villages_json", "[]").orEmpty()
        cycleNumber = prefs.getInt("current_cycle_number", 0)
        farmListCycleStartedAt = prefs.getLong("farm_cycle_started_at", 0L)
        resourceBuilderCycleStartedAt = prefs.getLong("resource_cycle_started_at", 0L)

        if (username.isBlank() || password.isBlank()) {
            logEvent("RECOVERY: credential database kosong/tidak valid; recovery dibatalkan")
            recoveringService = false
            return
        }

        running = true
        updateNotification("Farm Assistant — memulihkan service")
        logEvent("RECOVERY: proses Android dibuat ulang; memulihkan konfigurasi, WebView, dan scheduler")
        ensureServiceWebView()

        val cycleActive = prefs.getBoolean("cycle_active", false)
        val savedNextAt = prefs.getLong("next_run_at", 0L)
        val delay = savedNextAt - System.currentTimeMillis()

        handler.postDelayed({
            if (!running) return@postDelayed
            recoveringService = false
            if (cycleActive || delay <= 0L) {
                logEvent("RECOVERY: siklus terakhir belum selesai/interval sudah lewat; memulai ulang siklus")
                triggerScheduledCycle()
            } else {
                logEvent("RECOVERY: scheduler dipulihkan; run berikutnya dalam ${((delay + 999L) / 1000L)} detik")
                handler.removeCallbacks(nextRunRunnable)
                nextAt = savedNextAt
                handler.postDelayed(nextRunRunnable, delay)
                updateNextRun(delay)
                // Recovery mempertahankan urutan: countdown -> (30 detik kemudian)
                // Refresh Village -> countdown berakhir -> cycle.
                val savedCountdownStartedAt = prefs.getLong("countdown_started_at", (savedNextAt - delay).coerceAtLeast(0L))
                scheduleVillageRefreshForNextRun(savedCountdownStartedAt)
            }
        }, 800L)
    }

    private fun automationWebView(): WebView? {
        debugTrace("ENTER automationWebView")
        return visibleWebViewRef?.get() ?: webView
    }

    private fun handleVisiblePageFinished(url: String) {
        debugTrace("ENTER handleVisiblePageFinished")
        if (!running) return
        lastAutomationUrl = url
        val lower = url.lowercase(Locale.US)
        handlePageAfterConsent(url, lower, 0)
    }

    private fun handleLoginResultFromVisibleWebView(result: String) {
        debugTrace("ENTER handleLoginResultFromVisibleWebView")
        if (!running) return
        handler.post {
            if (!running) return@post
            when (result) {
                "submitting" -> updateNotification("Farm Assistant — mengirim login")
                "no_login_form" -> {
                    loginInProgress = false
                    reloginRequested = false
                    loginRetryCount = 0
                    logEvent("Session aktif terdeteksi; membuka Farm List")
                    handler.postDelayed({ triggerStartAllFarmLists() }, 250)
                }
                "no_username_field", "no_form" -> {
                    if (loginRetryCount < 20) handler.postDelayed({ autoLoginIfNeeded() }, 1000)
                    else {
                        loginInProgress = false
                        reloginRequested = false
                        logEvent("Form login Travian tidak dikenali")
                    }
                }
            }
        }
    }

    @SuppressLint("SetJavaScriptEnabled")
    private fun ensureServiceWebView() {
        debugTrace("ENTER ensureServiceWebView")
        if (webView != null) return
        webView = WebView(this@FarmAutomationService).apply {
            settings.javaScriptEnabled = true
            settings.domStorageEnabled = true
            settings.databaseEnabled = true
            settings.userAgentString =
                "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 " +
                "(KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
            settings.useWideViewPort = true
            settings.loadWithOverviewMode = true
            settings.textZoom = 100
            CookieManager.getInstance().setAcceptCookie(true)
            CookieManager.getInstance().setAcceptThirdPartyCookies(this, true)
            addJavascriptInterface(FarmBridge(), "AndroidFarm")
            webViewClient = object : WebViewClient() {
                override fun onPageFinished(view: WebView?, url: String?) {
                    debugTrace("ENTER onPageFinished")
                    super.onPageFinished(view, url)
                    if (url == null || !running) return
                    lastAutomationUrl = url
                    handlePageAfterConsent(url, url.lowercase(Locale.US), 0)
                }

                override fun onRenderProcessGone(view: WebView?, detail: android.webkit.RenderProcessGoneDetail?): Boolean {
                    debugTrace("ENTER onRenderProcessGone")
                    logEvent("RECOVERY: WebView renderer mati; membuat WebView baru")
                    if (view === webView) {
                        webView = null
                    }
                    webViewRecoveryInProgress = false
                    if (running) {
                        handler.postDelayed({ recoverWebView() }, 500L)
                    }
                    return true
                }
            }
        }
    }

    private fun onVisibleWebViewDetachedInternal() {
        debugTrace("ENTER onVisibleWebViewDetachedInternal")
        if (running && webView == null) ensureServiceWebView()
    }

    @SuppressLint("SetJavaScriptEnabled")
    private fun startAutomation() {
        debugTrace("ENTER startAutomation")

        // Jika tombol Bot diaktifkan kembali saat service/siklus lama masih aktif,
        // hentikan callback siklus lama terlebih dahulu agar tidak terjadi double cycle.
        if (running) {
            logEvent("Bot diaktifkan kembali — menghentikan callback siklus lama sebelum memulai siklus baru")
            handler.removeCallbacksAndMessages(null)
            builderInProgress = false
            pendingStartAll = false
            countdownCyclePending = false
            villageRefreshInProgress = false
            villageRefreshCompleted = false
            villageRefreshClosed = true
            farmListCycleComplete = false
            pendingBuilderResourceHref = ""
            builderVillageClickInProgress = false
            pendingUpgradeUrl = ""
            pendingUpgradeCosts = longArrayOf(0L, 0L, 0L, 0L)
            heroTransferCompleted = false
        }

        running = true
        cycleNumber = 0
        pendingStartAll = false
        loginInProgress = false
        reloginRequested = false
        loginRetryCount = 0
        startAllAttempt = 0
        consentAttempt = 0

        runCatching { CredentialDatabase(this).save(server, username, password) }
        getSharedPreferences(PREFS, MODE_PRIVATE).edit()
            .remove("server")
            .remove("username")
            .remove("password_secure")
            .putLong("interval_min_minutes", minMinutes)
            .putLong("interval_max_minutes", maxMinutes)
            .putBoolean("farm_list_enabled", farmListEnabled)
            .putBoolean("resource_builder_enabled", resourceBuilderEnabled)
            .putBoolean("service_running", true)
            .apply()

        logEvent("Background service dimulai. Range interval=${minMinutes}-${maxMinutes} menit; Farm List=${if (farmListEnabled) "ON" else "OFF"}; Resource Builder=${if (resourceBuilderEnabled) "ON" else "OFF"}; Village terpilih=${if (builderSelectionConfigured) selectedBuilderVillageIds.size else "SEMUA"}")
        updateNextRun(0L)
        updateNotification("Farm Assistant aktif — menyiapkan siklus")

        if (username.isBlank() || password.isBlank()) {
            logEvent("Background service gagal: username/password kosong")
            stopAutomation()
            return
        }

        if (webView == null && visibleWebViewRef?.get() == null) {
            ensureServiceWebView()
        }

        // Siklus pertama langsung dimulai. Refresh Village hanya dijalankan
        // pada fase countdown setelah cycle selesai.
        initialCyclePending = false
        countdownCyclePending = false
        villageRefreshInProgress = false
        villageRefreshCompleted = true
        villageRefreshClosed = true
        triggerScheduledCycle()
    }

    private fun triggerScheduledCycle() {
        debugTrace("ENTER triggerScheduledCycle")
        if (!running) return
        countdownCyclePending = false
        scheduledRefreshForNextRun = false
        val now = timeFormat.format(Date())
        cycleNumber += 1
        farmListCycleStartedAt = if (farmListEnabled) System.currentTimeMillis() else 0L
        resourceBuilderCycleStartedAt = 0L
        farmListCycleComplete = !farmListEnabled
        // Refresh Village adalah persiapan cycle berikutnya. Jika belum selesai
        // saat Next Run tiba, cycle menunggu sampai Refresh Village ditutup.
        getSharedPreferences(PREFS, MODE_PRIVATE).edit()
            .putString("last_run", now)
            .putInt("current_cycle_number", cycleNumber)
            .putBoolean("cycle_active", true)
            .putLong("farm_cycle_started_at", farmListCycleStartedAt)
            .putLong("resource_cycle_started_at", 0L)
            .apply()
        logEvent("CICLE START")
        handler.removeCallbacks(cycleWatchdogRunnable)
        // Refresh Village berjalan pada fase countdown. Jangan membatalkan timer-nya;
        // bila Next Run tiba lebih dulu, cycle akan menunggu refresh selesai.
        handler.postDelayed(cycleWatchdogRunnable, 15 * 60_000L)
        if (!villageRefreshClosed || villageRefreshInProgress || !villageRefreshCompleted) {
            logEvent("Siklus: menunggu AUTO REFRESH VILLAGE ditutup sebelum Farm List/Resource Builder")
            handler.postDelayed({ if (running) triggerScheduledCycle() }, 1_000L)
            return
        }
        triggerScheduledCycleActions()
    }

    private fun triggerScheduledCycleActions() {
        if (!running) return
        if (!villageRefreshCompleted) {
            logEvent("Siklus: menunggu REFRESH VILLAGE selesai")
            handler.postDelayed({ if (running) triggerScheduledCycleActions() }, 1_000L)
            return
        }
        if (farmListEnabled) {
            triggerStartAllFarmLists()
        } else if (resourceBuilderEnabled) {
            logEvent("Farm List OFF — menunggu AUTO REFRESH VILLAGE sebelum Resource Builder")
            maybeStartResourceBuilderAfterRefresh()
        } else {
            logEvent("Farm List OFF dan Resource Builder OFF — tidak ada aksi pada siklus ini")
            scheduleNextRandomRun()
        }
    }

    private fun triggerStartAllFarmLists() {
        debugTrace("ENTER triggerStartAllFarmLists")
        if (!running || !farmListEnabled) return
        pendingStartAll = true
        startAllAttempt = 0
        consentAttempt = 0
        updateNotification("Farm Assistant aktif — membuka Farm List")
        logEvent("Memulai siklus Start All Farm Lists")
        automationWebView()?.loadUrl("$server/build.php?id=39&gid=16&tt=99")
    }

    private fun handlePageAfterConsent(url: String, lower: String, attempt: Int): Unit {
        debugTrace("ENTER handlePageAfterConsent")
        if (!running) return
        acceptCookiesIfPresent { result ->
            if (!running) return@acceptCookiesIfPresent
            val consentStillVisible = result.contains("visible") || result.contains("clicked")
            if (consentStillVisible && attempt < 8) {
                consentAttempt = attempt + 1
                CookieManager.getInstance().flush()
                handler.postDelayed({ handlePageAfterConsent(url, lower, attempt + 1) }, 700)
                return@acceptCookiesIfPresent
            }

            if (lower.contains("gid=16") && lower.contains("tt=99")) {
                loginInProgress = false
                reloginRequested = false
                loginRetryCount = 0
                if (pendingStartAll) {
                    startAllAttempt = 0
                    handler.postDelayed({ clickStartAllFarmLists() }, 1200)
                }
                return@acceptCookiesIfPresent
            }

            if (villageRefreshInProgress && lower.contains("dorf1.php")) {
                handler.postDelayed({ inspectAutomaticVillageRefresh() }, 500L)
                return@acceptCookiesIfPresent
            }

            if (builderInProgress && lower.contains("dorf1.php")) {
                val expectedId = builderVillages.getOrNull(builderVillageIndex)?.first.orEmpty()

                if (builderVillages.isEmpty()) {
                    builderStage = "DISCOVER"
                    handler.postDelayed({ discoverVillagesForBuilder() }, 700)
                    return@acceptCookiesIfPresent
                }

                if (expectedId.isNotBlank() && pendingBuilderResourceHref.isNotBlank()) {
                    // Travian dapat redirect /dorf1.php?newdid=ID menjadi /dorf1.php.
                    // Karena itu verifikasi village aktif dari sidebar, sama seperti
                    // mekanisme REFRESH VILLAGE, lalu buka LinkResource dari database.
                    val expectedJson = JSONObject.quote(expectedId)
                    automationWebView()?.evaluateJavascript("""
                        (() => {
                            const expected = $expectedJson;
                            const urlId = location.href.match(/[?&]newdid=(\d+)/i)?.[1] || '';
                            const selectors = [
                                '#sidebarBoxVillagelist .listEntry.active',
                                '#sidebarBoxVillagelist .listEntry.selected',
                                '.villageList .listEntry.active',
                                '.villageList .listEntry.selected',
                                '[data-did].active'
                            ];
                            let active = null;
                            for (const selector of selectors) {
                                try { active = document.querySelector(selector); if (active) break; } catch (_) {}
                            }
                            const activeId = active?.getAttribute('data-did') || '';
                            const currentId = /^\d+$/.test(urlId) ? urlId : activeId;
                            return JSON.stringify({ok: currentId === expected, currentId, activeId, urlId});
                        })();
                    """.trimIndent()) { raw ->
                        val result = raw.orEmpty().trim('"').replace("\\\"", "\"")
                        val currentId = Regex("\"currentId\":\"(\\d*)\"").find(result)
                            ?.groupValues?.getOrNull(1).orEmpty()
                        if (result.contains("\"ok\":true")) {
                            if (builderStage != "OPEN_RESOURCE" && builderStage != "WAIT_UPGRADE" && builderStage != "ADVANCING") {
                                builderStage = "OPEN_RESOURCE"
                                logEvent("Resource Builder: village aktif benar ($expectedId); membuka Link Resource dari database")
                                handler.postDelayed({ openSavedBuilderResource() }, 300)
                            }
                        } else if (builderAttempt < 5) {
                            builderAttempt++
                            logEvent("Resource Builder: menunggu village aktif — expected=$expectedId current=${currentId.ifBlank { "-" }}; retry=$builderAttempt")
                            handler.postDelayed({
                                if (running && builderInProgress) {
                                    val saved = builderVillageLinks[expectedId].orEmpty().trim()
                                    val target = saved.ifBlank { "$server/dorf1.php?newdid=$expectedId" }
                                    automationWebView()?.loadUrl(absoluteBuilderHref(target))
                                }
                            }, 600)
                        } else {
                            logEvent("Resource Builder: gagal memastikan village aktif $expectedId; village dilewati")
                            builderAttempt = 0
                            pendingBuilderResourceHref = ""
                            goToNextBuilderVillage()
                        }
                    }
                    return@acceptCookiesIfPresent
                }

                if (builderStage == "LOAD_DORF" || builderStage == "CLICK_VILLAGE") {
                    builderStage = "CLICK_VILLAGE"
                    handler.postDelayed({ clickBuilderVillageFromDorf() }, 400)
                } else {
                    logEvent("Resource Builder: dorf1 menunggu village target; stage=$builderStage expected=$expectedId")
                }
                return@acceptCookiesIfPresent
            }
            if (builderInProgress && lower.contains("build.php") && !lower.contains("gid=16")) {
                if (builderStage == "TRANSFER_DONE") {
                    builderStage = "INSPECT_UPGRADE"
                    debugTrace("Resource Builder: kembali ke halaman resource setelah transfer -> inspectUpgradeResources()")
                    handler.postDelayed({ inspectUpgradeResources() }, 700)
                } else {
                    builderStage = "OPEN_TRANSFER"
                    pendingUpgradeUrl = automationWebView()?.url.orEmpty().ifBlank { "$server/build.php" }
                    debugTrace("Resource Builder: masuk halaman resource -> inspectUpgradeResources()")
                    handler.postDelayed({ inspectUpgradeResources() }, 700)
                }
                return@acceptCookiesIfPresent
            }


            if (isLikelyLoginPage(lower)) {
                clearVillageDatabaseOnLogout(url)
                if (username.isNotBlank() && password.isNotBlank()) {
                    loginInProgress = true
                    reloginRequested = true
                    loginRetryCount = 0
                    updateNotification("Farm Assistant — auto re-login")
                    logEvent("Session Travian habis; memulai auto re-login")
                    handler.postDelayed({ autoLoginIfNeeded() }, 500)
                } else {
                    logEvent("Session habis tetapi password tidak tersedia di RAM")
                }
                return@acceptCookiesIfPresent
            }

            if (loginInProgress) {
                handler.postDelayed({ autoLoginIfNeeded() }, 500)
            } else if (pendingStartAll) {
                detectLoginFormForScheduler()
            }
        }
    }

    private fun clearVillageDatabaseOnLogout(url: String) {
        val prefs = getSharedPreferences(PREFS, MODE_PRIVATE)
        val raw = prefs.getString("village_data_json", "[]").orEmpty()
        if (raw == "[]" || raw.isBlank()) return
        prefs.edit()
            .remove("village_data_json")
            .remove("resource_builder_targets_json")
            .remove("resource_builder_villages_json")
            .remove("resource_builder_selected_villages")
            .apply()
        builderVillages.clear()
        builderVillageLinks.clear()
        builderResourceLinks.clear()
        builderResourceLevels.clear()
        logEvent("LOGOUT/LOGIN TERDETEKSI — database village dihapus; url=$url")
    }

    private fun isLikelyLoginPage(url: String): Boolean {
        debugTrace("ENTER isLikelyLoginPage")
        return url.contains("login") || url.contains("logout") ||
            url.contains("anmelden") || url.contains("signin")
    }

    private fun autoLoginIfNeeded() {
        debugTrace("ENTER autoLoginIfNeeded")
        if (!running || !loginInProgress) return
        if (username.isBlank() || password.isBlank()) return
        loginRetryCount++
        if (loginRetryCount > 20) {
            loginInProgress = false
            reloginRequested = false
            logEvent("Auto re-login gagal setelah 20 percobaan")
            return
        }

        val usernameJson = JSONObject.quote(username)
        val passwordJson = JSONObject.quote(password)
        val js = """
            (() => {
                const username = $usernameJson;
                const password = $passwordJson;
                const visible = el => {
                    if (!el) return false;
                    const s = getComputedStyle(el);
                    return s.display !== 'none' && s.visibility !== 'hidden' && el.offsetParent !== null;
                };
                const inputs = [...document.querySelectorAll('input')].filter(visible);
                const passwordInput = inputs.find(x =>
                    (x.type || '').toLowerCase() === 'password' || /pass|password/i.test(x.name || '') || /pass|password/i.test(x.id || '')
                );
                if (!passwordInput) { AndroidFarm.onLoginResult('no_login_form'); return; }
                const userInput = inputs.find(x =>
                    /user|username|email|login|name/i.test(x.name || '') || /user|username|email|login|name/i.test(x.id || '') || (x.type || '').toLowerCase() === 'email'
                );
                if (!userInput) { AndroidFarm.onLoginResult('no_username_field'); return; }
                const setValue = (el, value) => {
                    const setter = Object.getOwnPropertyDescriptor(Object.getPrototypeOf(el), 'value')?.set;
                    if (setter) setter.call(el, value); else el.value = value;
                    el.dispatchEvent(new Event('input', {bubbles:true}));
                    el.dispatchEvent(new Event('change', {bubbles:true}));
                };
                setValue(userInput, username);
                setValue(passwordInput, password);
                const form = passwordInput.closest('form') || userInput.closest('form');
                if (!form) { AndroidFarm.onLoginResult('no_form'); return; }
                const buttons = [...form.querySelectorAll('button,input[type=submit],input[type=button],a')].filter(visible);
                const submitButton = buttons.find(x => /login|log in|sign in|anmelden|connexion|entrar|acceder/i.test((x.innerText || x.value || x.title || '').trim()));
                AndroidFarm.onLoginResult('submitting');
                if (submitButton) submitButton.click();
                else if (typeof form.requestSubmit === 'function') form.requestSubmit();
                else form.submit();
            })();
        """.trimIndent()
        automationWebView()?.evaluateJavascript(js, null)
    }

    private fun detectLoginFormForScheduler() {
        debugTrace("ENTER detectLoginFormForScheduler")
        automationWebView()?.evaluateJavascript("""
            (() => {
                const visible = el => {
                    if (!el) return false;
                    const s = getComputedStyle(el); const r = el.getBoundingClientRect();
                    return s.display !== 'none' && s.visibility !== 'hidden' && r.width > 0 && r.height > 0;
                };
                return [...document.querySelectorAll('input[type=password]')].some(visible) ? 'login_form' : 'not_login';
            })();
        """.trimIndent()) { raw ->
            if (raw.orEmpty().contains("login_form") && running) {
                loginInProgress = true
                reloginRequested = true
                loginRetryCount = 0
                logEvent("Form login terdeteksi saat scheduler berjalan")
                handler.postDelayed({ autoLoginIfNeeded() }, 250)
            }
        }
    }

    private fun clickStartAllFarmLists(): Unit {
        debugTrace("ENTER clickStartAllFarmLists")
        if (!running || !pendingStartAll) return
        val js = """
            (() => {
                const visible = el => {
                    if (!el) return false;
                    const s = getComputedStyle(el); const r = el.getBoundingClientRect();
                    return s.display !== 'none' && s.visibility !== 'hidden' && r.width > 0 && r.height > 0;
                };
                const norm = s => (s || '').replace(/\s+/g, ' ').trim().toLowerCase();
                const statusCount = () => {
                    let total = 0;
                    for (const el of document.querySelectorAll('#rallyPointFarmList .farmListStatus, .farmListStatus')) {
                        const m = norm(el.textContent).match(/(\d+)\s*\/\s*(\d+)/);
                        if (m) total += parseInt(m[1], 10);
                    }
                    return total;
                };
                const readyCount = () => {
                    let ready = 0;
                    for (const b of document.querySelectorAll('#rallyPointFarmList .farmListWrapper button.startFarmList, button.startFarmList')) {
                        if (visible(b) && !b.disabled && b.getAttribute('disabled') === null && b.getAttribute('aria-disabled') !== 'true') ready++;
                    }
                    return ready;
                };
                const dispatch = btn => {
                    btn.scrollIntoView({block:'center'});
                    // Gunakan native HTMLElement.click() terlebih dahulu. Beberapa
                    // handler Travian/jQuery tidak bereaksi terhadap MouseEvent buatan.
                    try { btn.click(); } catch (_) {}
                    // Event fallback untuk markup/handler lama.
                    try {
                        btn.dispatchEvent(new MouseEvent('mousedown', {bubbles:true, cancelable:true, view:window}));
                        btn.dispatchEvent(new MouseEvent('mouseup', {bubbles:true, cancelable:true, view:window}));
                    } catch (_) {}
                };
                const selectors = [
                    '#rallyPointFarmList button.startAllFarmLists',
                    'button.startAllFarmLists',
                    '.startAllFarmLists button',
                    '.startAllFarmLists'
                ];
                for (const selector of selectors) {
                    let nodes = [];
                    try { nodes = [...document.querySelectorAll(selector)]; } catch (_) {}
                    const btn = nodes.find(el => visible(el) && !el.disabled && el.getAttribute('aria-disabled') !== 'true');
                    if (btn) {
                        const before = statusCount();
                        const beforeReady = readyCount();
                        dispatch(btn);
                        return JSON.stringify({state:'clicked', before, beforeReady, selector});
                    }
                }
                const candidates = [...document.querySelectorAll('button,input[type=button],input[type=submit],a,[role=button]')];
                const textBtn = candidates.find(el => {
                    if (!visible(el) || el.disabled || el.getAttribute('aria-disabled') === 'true') return false;
                    const t = norm(el.innerText || el.textContent || el.value || el.title || el.getAttribute('aria-label'));
                    return /^(start all|start all farm lists?|start all farmlists?|send all)$/.test(t) || /start all.*farm/i.test(t);
                });
                if (textBtn) {
                    const before = statusCount();
                    const beforeReady = readyCount();
                    dispatch(textBtn);
                    return JSON.stringify({state:'clicked', before, beforeReady, selector:'text'});
                }
                return JSON.stringify({state:'not-found', before:statusCount(), beforeReady:readyCount()});
            })();
        """.trimIndent()
        automationWebView()?.evaluateJavascript(js) { raw ->
            val result = raw.orEmpty().trim('"').replace("\\\"", "\"")
            if (result.contains("\"state\":\"clicked\"")) {
                pendingStartAll = false
                startAllAttempt = 0
                raidVerificationAttempt = 0
                farmListProgressObserved = false
                farmListLastState = ""
                farmListStableChecks = 0
                fallbackFarmListMode = false
                raidCountBeforeStartAll = Regex("\"before\":(\\d+)").find(result)?.groupValues?.get(1)?.toIntOrNull() ?: 0
                farmListBeforeReady = Regex("\"readyBefore\":(\\d+)").find(result)?.groupValues?.get(1)?.toIntOrNull() ?: 0
                val now = timeFormat.format(Date())
                getSharedPreferences(PREFS, MODE_PRIVATE).edit().putString("last_run", now).apply()
                logEvent("Click Send All Farmlist Success")
                updateNotification("Farm List — menunggu 1 menit agar semua raid terkirim")
                
                // Send All adalah satu aksi dispatch. Tidak perlu polling status tombol
                // berulang-ulang karena tombol bisa tetap aktif walaupun semua request
                // sudah masuk. Beri Travian 60 detik untuk menyelesaikan seluruh dispatch,
                // lalu lanjut ke Resource Builder.
                handler.postDelayed({ finishFarmListAfterOneMinute() }, 60_000L)
            } else if (startAllAttempt < 10) {
                startAllAttempt++
                handler.postDelayed({ clickStartAllFarmLists() }, 1000)
            } else {
                pendingStartAll = false
                logEvent("Send All gagal: tombol tidak ditemukan setelah 10 percobaan; Farm List belum dianggap selesai")
                fallbackSequentialFarmListSend()
            }
        }
    }

    private fun finishFarmListAfterOneMinute() {
        debugTrace("ENTER finishFarmListAfterOneMinute")
        if (!running) return
        pendingStartAll = false
        fallbackFarmListMode = false
        val now = System.currentTimeMillis()
        if (farmListCycleStartedAt > 0L) {
            val farmDuration = (now - farmListCycleStartedAt).coerceAtLeast(0L)
            getSharedPreferences(PREFS, MODE_PRIVATE).edit()
                .putLong("farm_cycle_duration_ms", farmDuration)
                .putLong("farm_cycle_started_at", 0L)
                .apply()
            logEvent("Farm List: waktu proses ${formatDuration(farmDuration)}; jeda dispatch 60 detik selesai")
            farmListCycleStartedAt = 0L
        } else {
            logEvent("Farm List: jeda dispatch 60 detik selesai")
        }

        farmListCycleComplete = true
        maybeStartResourceBuilderAfterRefresh()
    }

    private fun verifyRaidDispatch(): Unit {
        debugTrace("ENTER verifyRaidDispatch")
        if (!running) return

        // Farm List adalah aksi dispatch, bukan proses yang harus ditunggu sampai
        // semua tombol Start menjadi disabled. Pada Travian tombol Start sering tetap
        // aktif walaupun request raid sudah berhasil dikirim. Verifikasi lama bisa
        // polling 20x + fallback + reload sampai watchdog 5 menit dan membuat
        // Resource Builder tidak pernah kebagian waktu.
        val js = """
            (() => {
                const visible = el => {
                    if (!el) return false;
                    const s = getComputedStyle(el), r = el.getBoundingClientRect();
                    return s.display !== 'none' && s.visibility !== 'hidden' && r.width > 0 && r.height > 0;
                };
                const norm = s => (s || '').replace(/\s+/g, ' ').trim().toLowerCase();
                let total = 0, wrappers = 0, ready = 0;
                for (const wrapper of document.querySelectorAll('#rallyPointFarmList .farmListWrapper')) {
                    wrappers++;
                    const status = wrapper.querySelector('.farmListStatus');
                    const m = norm(status?.textContent || '').match(/(\d+)\s*\/\s*(\d+)/);
                    if (m) total += parseInt(m[1], 10);
                    const btn = wrapper.querySelector('button.startFarmList');
                    if (btn && visible(btn) && !btn.disabled && btn.getAttribute('disabled') === null && btn.getAttribute('aria-disabled') !== 'true') ready++;
                }
                const allText = norm(document.querySelector('#rallyPointFarmList')?.innerText || '');
                const busy = /sending|loading|processing|mengirim|memproses/.test(allText);
                return JSON.stringify({total, wrappers, ready, busy});
            })();
        """.trimIndent()

        automationWebView()?.evaluateJavascript(js) { raw ->
            val result = raw.orEmpty().trim('"').replace("\\\"", "\"")
            val current = Regex("\"total\":(\\d+)").find(result)?.groupValues?.get(1)?.toIntOrNull() ?: 0
            val ready = Regex("\"ready\":(\\d+)").find(result)?.groupValues?.get(1)?.toIntOrNull() ?: 0
            val busy = Regex("\"busy\":(true|false)").find(result)?.groupValues?.get(1) == "true"
            val wrappers = Regex("\"wrappers\":(\\d+)").find(result)?.groupValues?.get(1)?.toIntOrNull() ?: 0

            if (current > raidCountBeforeStartAll || ready < farmListBeforeReady || busy) {
                farmListProgressObserved = true
            }

            // Beri AJAX Travian waktu singkat untuk mulai, tetapi jangan pernah
            // menahan siklus sampai menit ke-5 hanya karena tombol Start tetap aktif.
            val elapsedChecks = raidVerificationAttempt
            val dispatchSettled = wrappers > 0 && !busy &&
                (farmListProgressObserved || elapsedChecks >= 4)

            if (dispatchSettled || elapsedChecks >= 6) {
                val sent = (current - raidCountBeforeStartAll).coerceAtLeast(0)
                logEvent(
                    "Farm List selesai dispatch: $sent raid terdeteksi; " +
                        "tombol Start aktif=$ready; verifikasi=${elapsedChecks + 1}x — lanjut Resource Builder"
                )
                pendingStartAll = false
                fallbackFarmListMode = false
                farmListCycleComplete = true
                maybeStartResourceBuilderAfterRefresh()
            } else {
                raidVerificationAttempt++
                logEvent(
                    "Farm List verifikasi ${raidVerificationAttempt}/6; raid=$current; " +
                        "tombol Start aktif=$ready; busy=$busy; progress=${if (farmListProgressObserved) "YA" else "BELUM"}"
                )
                updateNotification("Farm List — dispatch ${raidVerificationAttempt}/6")
                handler.postDelayed({ verifyRaidDispatch() }, 1000)
            }
        }
    }

    private fun fallbackSequentialFarmListSend() {
        debugTrace("ENTER fallbackSequentialFarmListSend")
        if (!running) return
        updateNotification("Farm List — fallback, menyelesaikan pengiriman")
        val js = """
            (() => {
                const visible = el => {
                    if (!el) return false;
                    const s = getComputedStyle(el), r = el.getBoundingClientRect();
                    return s.display !== 'none' && s.visibility !== 'hidden' && r.width > 0 && r.height > 0;
                };
                const statusCount = () => {
                    let total = 0;
                    for (const wrapper of document.querySelectorAll('#rallyPointFarmList .farmListWrapper')) {
                        const m = (wrapper.querySelector('.farmListStatus')?.textContent || '').replace(/\s+/g,' ').match(/(\d+)\s*\/\s*(\d+)/);
                        if (m) total += parseInt(m[1],10);
                    }
                    return total;
                };
                const buttons = [...document.querySelectorAll('#rallyPointFarmList .farmListWrapper button.startFarmList, button.startFarmList')]
                    .filter(b => visible(b) && !b.disabled && b.getAttribute('disabled') === null && b.getAttribute('aria-disabled') !== 'true');
                const readyBefore = buttons.length;
                const totalBefore = statusCount();
                for (const btn of buttons) {
                    btn.scrollIntoView({block:'center'});
                    try { btn.click(); } catch (_) {}
                    try {
                        btn.dispatchEvent(new MouseEvent('mousedown', {bubbles:true, cancelable:true, view:window}));
                        btn.dispatchEvent(new MouseEvent('mouseup', {bubbles:true, cancelable:true, view:window}));
                    } catch (_) {}
                }
                return JSON.stringify({clicked:buttons.length, readyBefore, totalBefore});
            })();
        """.trimIndent()
        automationWebView()?.evaluateJavascript(js) { raw ->
            val result = raw.orEmpty().trim('"').replace("\\\"", "\"")
            val clicked = Regex("\"clicked\":(\\d+)").find(result)?.groupValues?.get(1)?.toIntOrNull() ?: 0
                farmListBeforeReady = Regex("\"readyBefore\":(\\d+)").find(result)?.groupValues?.get(1)?.toIntOrNull() ?: 0
            farmListProgressObserved = false
            farmListLastState = ""
            farmListStableChecks = 0
            fallbackFarmListMode = true
            logEvent("Fallback Start per Farm List: $clicked tombol diklik; sebelum=$farmListBeforeReady tombol siap; menunggu 60 detik")
            raidVerificationAttempt = 0
            updateNotification("Farm List — fallback, menunggu 1 menit")
            handler.postDelayed({ finishFarmListAfterOneMinute() }, 60_000L)
        }
    }

    private fun verifyFallbackRaidCompletion(): Unit {
        debugTrace("ENTER verifyFallbackRaidCompletion")
        if (!running) return

        // Fallback hanya memastikan request sudah diberi kesempatan diproses.
        // Jangan reload Farm List berulang-ulang: reload + polling lama dapat
        // menghabiskan seluruh watchdog dan mencegah Resource Builder berjalan.
        val js = """
            (() => {
                const norm = s => (s || '').replace(/\s+/g, ' ').trim().toLowerCase();
                let total = 0, wrappers = 0;
                for (const wrapper of document.querySelectorAll('#rallyPointFarmList .farmListWrapper')) {
                    wrappers++;
                    const m = norm(wrapper.querySelector('.farmListStatus')?.textContent || '').match(/(\d+)\s*\/\s*(\d+)/);
                    if (m) total += parseInt(m[1], 10);
                }
                const allText = norm(document.querySelector('#rallyPointFarmList')?.innerText || '');
                const busy = /sending|loading|processing|mengirim|memproses/.test(allText);
                return JSON.stringify({total, wrappers, busy});
            })();
        """.trimIndent()

        automationWebView()?.evaluateJavascript(js) { raw ->
            val result = raw.orEmpty().trim('"').replace("\\\"", "\"")
            val current = Regex("\"total\":(\\d+)").find(result)?.groupValues?.get(1)?.toIntOrNull() ?: 0
            val wrappers = Regex("\"wrappers\":(\\d+)").find(result)?.groupValues?.get(1)?.toIntOrNull() ?: 0
            val busy = Regex("\"busy\":(true|false)").find(result)?.groupValues?.get(1) == "true"

            raidVerificationAttempt++
            if (busy) farmListProgressObserved = true
            val sent = (current - raidCountBeforeStartAll).coerceAtLeast(0)

            // Maksimal 6 detik. Jika status raid tidak berubah, tetap lanjut karena
            // tujuan fallback adalah dispatch, bukan menunggu counter berubah.
            if (!busy && (farmListProgressObserved || raidVerificationAttempt >= 3) || raidVerificationAttempt >= 6) {
                logEvent(
                    "Fallback Farm List selesai dispatch: $sent raid terdeteksi; " +
                        "wrappers=$wrappers; lanjut Resource Builder"
                )
                fallbackFarmListMode = false
                pendingStartAll = false
                farmListCycleComplete = true
                maybeStartResourceBuilderAfterRefresh()
            } else {
                logEvent("Fallback Farm List menunggu dispatch ${raidVerificationAttempt}/6; raid=$current; busy=$busy")
                updateNotification("Farm List — fallback ${raidVerificationAttempt}/6")
                handler.postDelayed({ verifyFallbackRaidCompletion() }, 1000)
            }
        }
    }

    /**
     * Resource Builder: setelah raid berhasil dijalankan, kunjungi setiap village
     * dan upgrade satu resource field dengan level terendah yang tersedia.
     * Strategi ini sengaja hanya melakukan satu upgrade per village per siklus.
     */
    private fun maybeStartResourceBuilderAfterRefresh() {
        debugTrace("ENTER maybeStartResourceBuilderAfterRefresh")
        if (!running) return
        if (countdownCyclePending && !getSharedPreferences(PREFS, MODE_PRIVATE).getBoolean("cycle_active", false)) {
            logEvent("Resource Builder: refresh dijalankan untuk cycle berikutnya; menunggu countdown berakhir")
            return
        }
        if (!resourceBuilderEnabled) {
            scheduleNextRandomRun()
            return
        }
        if (!farmListCycleComplete) {
            logEvent("Resource Builder: menunggu Farm List selesai")
            return
        }
        if (!villageRefreshCompleted) {
            logEvent("Resource Builder: menunggu REFRESH VILLAGE selesai")
            updateNotification("Menunggu REFRESH VILLAGE sebelum Resource Builder")
            return
        }
        startResourceBuilderCycle()
    }

    private fun startAutomaticVillageRefresh() {
        debugTrace("ENTER startAutomaticVillageRefresh")
        if (!running || villageRefreshInProgress || villageRefreshCompleted) return
        val records = loadVillageDataRecordsFromPrefs().filter { it.isChecklist && it.id.isNotBlank() }
        if (records.isEmpty()) {
            villageRefreshCompleted = true
            villageRefreshInProgress = false
            villageRefreshClosed = true
            logEvent("AUTO REFRESH VILLAGE: tidak ada village checklist; refresh dianggap selesai")
            if (initialCyclePending) {
                initialCyclePending = false
                countdownCyclePending = false
                triggerScheduledCycle()
            } else {
                maybeStartResourceBuilderAfterRefresh()
            }
            return
        }
        ensureServiceWebView()
        villageRefreshVillages = records.map { it.id to it.namaVillage }.toMutableList()
        villageRefreshIndex = 0
        villageRefreshRetry = 0
        villageRefreshInspectInFlight = false
        villageRefreshInProgress = true
        villageRefreshCompleted = false
        villageRefreshClosed = false
        villageRefreshStartedAt = System.currentTimeMillis()
        villageRefreshTimeoutRunnable?.let { handler.removeCallbacks(it) }
        villageRefreshTimeoutRunnable = Runnable {
            if (running && villageRefreshInProgress) {
                closeAutomaticVillageRefresh("TIMEOUT 3 MENIT — refresh ditutup paksa")
            }
        }.also { handler.postDelayed(it, 180_000L) }
        val selectedIds = villageRefreshVillages.map { it.first }.toSet()
        val cleared = loadVillageDataRecordsFromPrefs().map {
            if (it.id in selectedIds) it.copy(linkResource = "", minLvl = -1) else it
        }
        saveVillageDataRecordsForService(cleared)
        logEvent("REFRESH VILLAGE START")
        updateNotification("Refresh Village — 0/${villageRefreshVillages.size}")
        loadNextAutomaticVillageRefresh()
    }

    private fun loadNextAutomaticVillageRefresh() {
        if (!running || !villageRefreshInProgress) return
        if (villageRefreshIndex >= villageRefreshVillages.size) {
            villageRefreshRetry = 0
            logEvent("REFRESH VILLAGE END")
            closeAutomaticVillageRefresh("semua village checklist selesai")
            return
        }
        val (id, name) = villageRefreshVillages[villageRefreshIndex]
        villageRefreshRetry = 0
        villageRefreshInspectInFlight = false
        updateNotification("Refresh Village — ${villageRefreshIndex + 1}/${villageRefreshVillages.size}: $name")
        logEvent("AUTO REFRESH VILLAGE: [${villageRefreshIndex + 1}/${villageRefreshVillages.size}] membuka $name (ID $id)")
        automationWebView()?.loadUrl("$server/dorf1.php?newdid=$id")
    }

    /**
     * Refresh otomatis harus memakai aturan identifikasi field yang sama dengan
     * REFRESH VILLAGE setelah login di MainActivity:
     * - ID, GID dan level harus berasal dari field yang sama.
     * - Tidak boleh default gid=1.
     * - Target resource disimpan lengkap (LinkResource, ResourceId, ResourceGid, MinLvl).
     * Dengan begitu refresh otomatis tidak menghasilkan database yang berbeda
     * dengan refresh manual/setelah login.
     */
    private fun inspectAutomaticVillageRefresh() {
        if (!running || !villageRefreshInProgress || villageRefreshInspectInFlight) return
        val pair = villageRefreshVillages.getOrNull(villageRefreshIndex) ?: return
        villageRefreshInspectInFlight = true
        val expectedId = pair.first
        val expectedName = pair.second
        val idJson = JSONObject.quote(expectedId)

        val js = """
            (() => {
                const expectedId = $idJson;
                const clean = s => String(s || '').replace(/\s+/g,' ').trim();
                const url = location.href;
                const match = url.match(/[?&]newdid=(\d+)/i);
                let currentId = match ? match[1] : '';

                const activeCandidates = [
                    '#sidebarBoxVillagelist .listEntry.active',
                    '#sidebarBoxVillagelist .listEntry.selected',
                    '.villageList .listEntry.active',
                    '.villageList .listEntry.selected',
                    '[data-did].active'
                ];
                let active = null;
                for (const selector of activeCandidates) {
                    try { active = document.querySelector(selector); if (active) break; } catch (_) {}
                }
                const activeId = active?.getAttribute('data-did') || '';
                const activeName = clean(active?.querySelector('.name')?.textContent || '');
                if (!currentId && /^\d+$/.test(activeId)) currentId = activeId;

                if (currentId !== expectedId) {
                    return JSON.stringify({
                        ready:false, reason:'WRONG_VILLAGE', id:currentId, expectedId,
                        url, activeId, activeName, readyState:document.readyState
                    });
                }

                const container = document.querySelector('#resourceFieldContainer');
                if (!container) {
                    return JSON.stringify({
                        ready:false, reason:'NO_RESOURCE_CONTAINER', id:currentId, expectedId,
                        url, activeId, activeName, readyState:document.readyState
                    });
                }

                const attr = (el, names) => {
                    for (const name of names) {
                        const value = el?.getAttribute?.(name);
                        if (value != null && String(value).trim() !== '') return String(value).trim();
                    }
                    return '';
                };

                const numberFrom = (value, patterns) => {
                    const text = String(value || '');
                    for (const pattern of patterns) {
                        const m = text.match(pattern);
                        if (m) return parseInt(m[1], 10);
                    }
                    return -1;
                };

                const readFieldValue = (anchor, names, patterns, maxDepth = 10) => {
                    let node = anchor;
                    for (let depth = 0; depth < maxDepth && node; depth++, node = node.parentElement) {
                        const className = typeof node.className === 'string' ? node.className : '';
                        const values = [
                            ...names.map(name => node.getAttribute?.(name) || ''),
                            node.getAttribute?.('title') || '',
                            node.getAttribute?.('aria-label') || '',
                            className
                        ];
                        const value = numberFrom(values.join(' '), patterns);
                        if (value >= 0) return value;
                    }
                    return -1;
                };

                const fieldAnchors = [
                    ...container.querySelectorAll(
                        'a[href*="build.php?id="], a[data-id], a[id], .buildingSlot a, .resourceField a'
                    )
                ];

                const candidates = [];
                const seen = new Set();

                for (const a of fieldAnchors) {
                    const hrefRaw = a.getAttribute('href') || '';
                    const absoluteHref = (() => {
                        try { return new URL(hrefRaw, location.href); } catch (_) { return null; }
                    })();

                    let fieldId = absoluteHref?.searchParams.get('id')
                        ? parseInt(absoluteHref.searchParams.get('id'), 10) : -1;
                    if (!(fieldId >= 1 && fieldId <= 18)) {
                        fieldId = readFieldValue(
                            a,
                            ['data-id', 'data-field-id', 'data-fieldid'],
                            [
                                /(?:^|[\s_-])id\s*([0-9]{1,2})(?=$|[\s_-])/i,
                                /(?:^|[\s_-])field(?:id)?\s*([0-9]{1,2})(?=$|[\s_-])/i
                            ]
                        );
                    }
                    if (!(fieldId >= 1 && fieldId <= 18) || seen.has(fieldId)) continue;

                    // GID wajib diambil dari field yang sama; jangan default gid=1.
                    let gid = absoluteHref?.searchParams.get('gid')
                        ? parseInt(absoluteHref.searchParams.get('gid'), 10) : -1;
                    if (!(gid >= 1 && gid <= 4)) {
                        gid = readFieldValue(
                            a,
                            ['data-gid', 'data-building-gid', 'data-buildingid', 'data-building-id'],
                            [
                                /(?:^|[\s_-])gid\s*([1-4])(?=$|[\s_-])/i,
                                /(?:^|[\s_-])building(?:id|gid)?\s*([1-4])(?=$|[\s_-])/i
                            ]
                        );
                    }
                    if (!(gid >= 1 && gid <= 4)) continue;

                    const level = readFieldValue(
                        a,
                        ['data-level', 'data-lvl', 'data-field-level'],
                        [
                            /(?:^|[\s_-])(?:a)?level\s*([0-9]{1,2})(?=$|[\s_-])/i,
                            /(?:^|[\s_-])lvl\s*([0-9]{1,2})(?=$|[\s_-])/i,
                            /(?:^|[\s_-])level([0-9]{1,2})(?=$|[\s_-])/i,
                            /(?:^|[\s_-])lvl([0-9]{1,2})(?=$|[\s_-])/i
                        ]
                    );
                    if (!(level >= 0)) continue;

                    const disabled = a.classList.contains('disabled') ||
                        !!a.closest('.disabled') ||
                        a.getAttribute('aria-disabled') === 'true' ||
                        a.getAttribute('data-disabled') === 'true';

                    if (absoluteHref) {
                        absoluteHref.searchParams.set('gid', String(gid));
                        absoluteHref.searchParams.set('newdid', expectedId);
                    }

                    seen.add(fieldId);
                    candidates.push({
                        fieldId, gid, level,
                        href: absoluteHref?.href || hrefRaw,
                        disabled
                    });
                }

                candidates.sort((a,b) =>
                    a.level - b.level || a.fieldId - b.fieldId
                );

                const lowest = candidates.find(
                    x => !x.disabled && x.level >= 0 && x.gid >= 1 && x.gid <= 4 && x.level < 10
                ) || null;

                // Samakan syarat readiness dengan refresh setelah login:
                // minimal 18 field harus lengkap sebagai pasangan ID+GID+level.
                const resourceFieldsComplete = candidates.length >= 18;
                if (!resourceFieldsComplete) {
                    return JSON.stringify({
                        ready:false, reason:'FIELDS_NOT_READY',
                        id:currentId, expectedId, url, activeId, activeName,
                        fieldCount:candidates.length
                    });
                }

                const entry = [...document.querySelectorAll('[data-did]')]
                    .find(e => String(e.getAttribute('data-did') || '') === expectedId);
                const name = clean(
                    entry?.querySelector('.name')?.textContent ||
                    entry?.querySelector('[class*="name"]')?.textContent ||
                    activeName || expectedName || ''
                );

                return JSON.stringify({
                    ready:true,
                    id:expectedId,
                    name,
                    minLevel:lowest?.level ?? Math.min(...candidates.map(x => x.level)),
                    lowest
                });
            })();
        """.trimIndent()

        automationWebView()?.evaluateJavascript(js) { raw ->
            val result = raw.orEmpty().trim().trim('"').replace("\\\"", "\"")
            val json = runCatching { JSONObject(result) }.getOrNull()
            villageRefreshInspectInFlight = false

            if (json?.optBoolean("ready", false) != true) {
                villageRefreshRetry++
                val reason = json?.optString("reason", "NOT_READY") ?: "NOT_READY"
                if (villageRefreshRetry <= 12) {
                    if (villageRefreshRetry == 1 || villageRefreshRetry == 8) {
                        logEvent(
                            "AUTO REFRESH VILLAGE: $expectedName belum siap " +
                                "reason=$reason retry=$villageRefreshRetry"
                        )
                    }
                    handler.postDelayed({ inspectAutomaticVillageRefresh() }, 800L)
                } else {
                    logEvent("AUTO REFRESH VILLAGE: $expectedName timeout; village dilewati")
                    villageRefreshIndex++
                    handler.postDelayed({ loadNextAutomaticVillageRefresh() }, 500L)
                }
                return@evaluateJavascript
            }

            val lowest = json.optJSONObject("lowest")
            val href = lowest?.optString("href").orEmpty().trim()
            val resourceId = lowest?.optInt("fieldId", -1) ?: -1
            val resourceGid = lowest?.optInt("gid", -1) ?: -1
            val minLevel = lowest?.optInt("level", json.optInt("minLevel", -1))
                ?: json.optInt("minLevel", -1)

            val records = loadVillageDataRecordsFromPrefs().toMutableList()
            val pos = records.indexOfFirst { it.id == expectedId }

            if (minLevel >= 10) {
                if (pos >= 0) {
                    records.removeAt(pos)
                    saveVillageDataRecordsForService(records)
                }
                logEvent(
                    "AUTO REFRESH VILLAGE: $expectedName dihapus dari DATABASE — " +
                        "MinLvl=L$minLevel (>=10)"
                )
            } else if (
                pos >= 0 &&
                href.isNotBlank() &&
                resourceId in 1..18 &&
                resourceGid in 1..4 &&
                minLevel >= 0
            ) {
                val old = records[pos]
                records[pos] = old.copy(
                    namaVillage = json.optString("name").trim().ifBlank { expectedName },
                    linkVillage = "$server/dorf1.php?newdid=$expectedId",
                    linkResource = href,
                    resourceId = resourceId.toString(),
                    resourceGid = resourceGid.toString(),
                    minLvl = minLevel
                )
                saveVillageDataRecordsForService(records)

                logEvent("Village $expectedName Updated min L$minLevel")
            } else {
                logEvent(
                    "AUTO REFRESH VILLAGE: $expectedName target tidak valid — " +
                        "min=L$minLevel id=$resourceId gid=$resourceGid"
                )
            }

            villageRefreshIndex++
            handler.postDelayed({ loadNextAutomaticVillageRefresh() }, 500L)
        }
    }

    private fun saveVillageDataRecordsForService(records: List<VillageDataRecord>) {
        val array = org.json.JSONArray()
        records.distinctBy { it.id }.forEach { item ->
            array.put(JSONObject().apply {
                put("IsChecklist", item.isChecklist)
                put("NamaVillage", item.namaVillage)
                put("Id", item.id)
                put("LinkVillage", item.linkVillage)
                put("LinkResource", item.linkResource)
                put("ResourceId", item.resourceId)
                put("ResourceGid", item.resourceGid)
                put("MinLvl", item.minLvl)
            })
        }
        getSharedPreferences(PREFS, MODE_PRIVATE).edit().putString("village_data_json", array.toString()).apply()
    }

    private fun startResourceBuilderCycle() {
        debugTrace("ENTER startResourceBuilderCycle")
        if (!running) return
        builderInProgress = true
        builderVillages.clear()
        builderVillageIndex = 0
        builderAttempt = 0
        pendingBuilderResourceHref = ""
        builderVillageClickInProgress = false
        builderDiscoverInFlight = false
        // Village Data adalah single source of truth. Resource Builder hanya memproses
        // record yang IsChecklist=true dan memakai LinkVillage + LinkResource yang
        // sudah disimpan oleh Auto Refresh. Tidak ada rediscovery target resource di sini.
        val now = System.currentTimeMillis()
        if (farmListCycleStartedAt > 0L) {
            val farmDuration = (now - farmListCycleStartedAt).coerceAtLeast(0L)
            getSharedPreferences(PREFS, MODE_PRIVATE).edit()
                .putLong("farm_cycle_duration_ms", farmDuration)
                .putLong("farm_cycle_started_at", 0L)
                .apply()
            logEvent("Farm List: waktu proses ${formatDuration(farmDuration)}")
            farmListCycleStartedAt = 0L
        }
        resourceBuilderCycleStartedAt = now
        getSharedPreferences(PREFS, MODE_PRIVATE).edit()
            .putLong("resource_cycle_started_at", now)
            .apply()
        val hasVillageData = loadBuilderStateFromVillageData()
        if (!hasVillageData) {
            logEvent("Resource Builder: tidak ada village yang dicentang; siklus selesai")
            finishResourceBuilderCycle()
            return
        }

        val missingTargets = builderVillages.filter { builderResourceLinks[it.first].isNullOrBlank() }
        if (missingTargets.isNotEmpty()) {
            logEvent(
                "Resource Builder: target resource belum tersimpan untuk " +
                    missingTargets.joinToString(" | ") { "${it.second} [${it.first}]" } +
                    "; village tersebut dilewati"
            )
            builderVillages = builderVillages.filter { builderResourceLinks[it.first].orEmpty().isNotBlank() }.toMutableList()
        }

        if (builderVillages.isEmpty()) {
            finishResourceBuilderCycle()
            return
        }

        builderStage = "LOAD_DORF"
        updateNotification("Farm Assistant — Resource Builder menyiapkan village")
        logEvent(
            "Resource Builder: ${builderVillages.size} village siap; " +
                "menggunakan link village + target resource yang sudah tersimpan"
        )
        // Reset watchdog saat masuk fase Builder agar timeout Farm List tidak
        // mematikan Builder yang memang membutuhkan waktu lebih dari 1 menit.
        handler.removeCallbacks(cycleWatchdogRunnable)
        handler.postDelayed(cycleWatchdogRunnable, 15 * 60_000L)
        automationWebView()?.loadUrl("$server/dorf1.php")
    }

    private fun processResourceBuilderVillage() {
        debugTrace("ENTER processResourceBuilderVillage")
        if (!running || !builderInProgress) return

        if (builderVillageIndex >= builderVillages.size) {
            finishResourceBuilderCycle()
            return
        }

        val (villageId, villageName) = builderVillages[builderVillageIndex]
        val resourceHref = builderResourceLinks[villageId].orEmpty()
        if (resourceHref.isBlank()) {
            logEvent("Resource Builder: target resource belum tersimpan untuk $villageName (ID $villageId); village dilewati. Jalankan REFRESH VILLAGE terlebih dahulu.")
            goToNextBuilderVillage()
            return
        }

        builderAttempt = 0
        pendingBuilderResourceHref = resourceHref
        builderVillageClickInProgress = false
        builderStage = "WAIT_VILLAGE"
        val savedLevel = builderResourceLevels[villageId]
        val savedVillageHref = builderVillageLinks[villageId].orEmpty().trim()
        val villageUrl = if (savedVillageHref.isNotBlank() &&
            Regex("[?&]newdid=${Regex.escape(villageId)}(?:&|$)", RegexOption.IGNORE_CASE).containsMatchIn(savedVillageHref)) {
            absoluteBuilderHref(savedVillageHref)
        } else {
            "$server/dorf1.php?newdid=$villageId"
        }
        saveDebugResourceBuilderVillageLink(villageId, savedVillageHref.ifBlank { villageUrl })
        logEvent(
            "Resource Builder: village ${builderVillageIndex + 1}/${builderVillages.size} — $villageName (ID $villageId); " +
                "LINK VILLAGE=$villageUrl; target=${resourceHref}${savedLevel?.let { "; level=L$it" } ?: ""}"
        )
        updateNotification("Resource Builder — ${builderVillageIndex + 1}/${builderVillages.size}: $villageName")

        // Tidak lagi rediscovery/klik sidebar. Link Village sudah disimpan saat
        // REFRESH VILLAGE dan sekarang dipakai langsung untuk berpindah context.
        automationWebView()?.loadUrl(villageUrl)
    }

    private fun openSavedBuilderResource(): Unit {
        debugTrace("ENTER openSavedBuilderResource")
        if (!running || !builderInProgress || pendingBuilderResourceHref.isBlank()) return
        val (villageId, villageName) = builderVillages.getOrNull(builderVillageIndex) ?: return
        var href = absoluteBuilderHref(pendingBuilderResourceHref)
        if (!Regex("[?&]newdid=\\d+", RegexOption.IGNORE_CASE).containsMatchIn(href)) {
            href += if (href.contains("?")) "&newdid=$villageId" else "?newdid=$villageId"
        }
        builderStage = "OPEN_RESOURCE"
        builderVillageClickInProgress = false
        val fieldId = Regex("[?&]id=(\\d+)", RegexOption.IGNORE_CASE).find(href)?.groupValues?.getOrNull(1).orEmpty()
        logEvent("Resource Builder: $villageName — masuk langsung ke target resource id=$fieldId href=$href")
        automationWebView()?.loadUrl(href)
    }

    private fun saveDebugResourceBuilderVillageLink(villageId: String, savedVillageHref: String = "") {
        val prefs = getSharedPreferences(PREFS, MODE_PRIVATE)
        val targetUrl = savedVillageHref.trim().ifBlank { "$server/dorf1.php?newdid=$villageId" }
        prefs.edit()
            .putString("debug_last_resource_builder_village_link", targetUrl)
            .apply()
    }

    private fun clickBuilderVillageFromDorf(): Unit {
        debugTrace("ENTER clickBuilderVillageFromDorf")
        if (!running || !builderInProgress) return
        val village = builderVillages.getOrNull(builderVillageIndex) ?: return
        val savedVillageHref = builderVillageLinks[village.first].orEmpty().trim()
        val targetUrl = if (savedVillageHref.isNotBlank() &&
            Regex("[?&]newdid=${Regex.escape(village.first)}(?:&|$)", RegexOption.IGNORE_CASE).containsMatchIn(savedVillageHref)) {
            absoluteBuilderHref(savedVillageHref)
        } else {
            "$server/dorf1.php?newdid=${village.first}"
        }
        saveDebugResourceBuilderVillageLink(village.first, savedVillageHref.ifBlank { targetUrl })
        pendingBuilderResourceHref = builderResourceLinks[village.first].orEmpty()
        builderVillageClickInProgress = false
        builderStage = "WAIT_VILLAGE"
        logEvent("Resource Builder: menggunakan LINK VILLAGE tersimpan untuk ${village.second} (ID ${village.first}) — $targetUrl")
        automationWebView()?.loadUrl(targetUrl)
    }

    private fun inspectUpgradeResources(): Unit {
        debugTrace("ENTER inspectUpgradeResources")
        if (!running || !builderInProgress) return

        val currentUrl = automationWebView()?.url.orEmpty()
        if (!currentUrl.contains("build.php", ignoreCase = true) ||
            currentUrl.contains("gid=16", ignoreCase = true)) {
            logEvent("Resource Builder: halaman target bukan build.php; URL=$currentUrl; membuka ulang target tersimpan")
            builderStage = "OPEN_RESOURCE"
            handler.postDelayed({ openSavedBuilderResource() }, 400L)
            return
        }

        builderStage = "INSPECT_UPGRADE"

        // Beri waktu 3 detik agar seluruh DOM/komponen halaman resource selesai dirender
        // sebelum menentukan jalur direct upgrade atau Hero Transfer.
        logEvent("Resource Builder: menunggu 3 detik agar DOM halaman resource selesai dimuat")
        handler.postDelayed({
            if (!running || !builderInProgress) return@postDelayed
            if (automationWebView()?.url.orEmpty().contains("gid=16", ignoreCase = true)) {
                logEvent("Resource Builder: halaman berubah ke Farm List; pemeriksaan Upgrade dibatalkan")
                return@postDelayed
            }
            inspectUpgradeResourcesAfterDomReady()
        }, 3_000L)
    }

    private fun inspectUpgradeResourcesAfterDomReady(): Unit {
        debugTrace("ENTER inspectUpgradeResourcesAfterDomReady")
        if (!running || !builderInProgress) return

        // ALUR UTAMA YANG DIMINTA:
        // Setelah DOM siap, cukup cek SELURUH TEKS HALAMAN.
        // Tidak peduli "Upgrade to level" berada di button, div, link, atau elemen lain.
        //
        // ADA    -> langsung klik Upgrade.
        // TIDAK ADA -> jalur Hero lama yang sudah terbukti berhasil:
        //              buka resource Hero -> Transfer Selected -> verifikasi
        //              -> Upgrade.
        //
        // Jangan gunakan perhitungan biaya/resource sebagai penentu cabang,
        // karena Travian sendiri sudah menentukan ketersediaan upgrade lewat
        // tombol Upgrade to level.
        val js = """
            (() => {
                const pageText = String(document.body?.innerText || document.documentElement?.innerText || '');
                const normalizedText = pageText.replace(/\s+/g, ' ').trim();
                const hasUpgradeText = /upgrade\s+to\s+level/i.test(normalizedText);

                return JSON.stringify({
                    state: hasUpgradeText ? 'upgrade_available' : 'hero_required',
                    hasUpgradeText: hasUpgradeText,
                    matchedText: hasUpgradeText ? (normalizedText.match(/upgrade\s+to\s+level[^\n]*/i)?.[0] || 'Upgrade to level') : '',
                    pageTextLength: normalizedText.length
                });
            })();
        """.trimIndent()

        automationWebView()?.evaluateJavascript(js) { raw ->
            val result = raw.orEmpty().trim('"').replace("\\\"", "\"")

            if (result.contains("\"state\":\"upgrade_available\"")) {
                logEvent("Resource Builder: halaman mengandung teks 'Upgrade to level' — langsung klik Upgrade")
                heroTransferCompleted = false
                pendingUpgradeCosts = longArrayOf(0L, 0L, 0L, 0L)
                clickResourceUpgrade()
            } else {
                logEvent("Resource Builder: halaman tidak mengandung teks 'Upgrade to level' — masuk jalur Hero Transfer")
                pendingUpgradeCosts = longArrayOf(0L, 0L, 0L, 0L)
                pendingUpgradeUrl = automationWebView()?.url.orEmpty().ifBlank { "$server/build.php" }
                inventoryUseAttempt = 0
                updateNotification("Resource Builder — transfer resource Hero")
                clickRedResourceForTransfer()
            }
        }
    }

    private fun clickRedResourceForTransfer() {
        debugTrace("ENTER clickRedResourceForTransfer()")

        if (!running || !builderInProgress) {
            debugTrace(
                "HERO TRANSFER: batal — running=$running builderInProgress=$builderInProgress"
            )
            return
        }

        if (pendingUpgradeUrl.isBlank()) {
            pendingUpgradeUrl = automationWebView()?.url.orEmpty()
                .ifBlank { "$server/build.php" }
        }

        val js = """
            (() => {
                const visible = el => {
                    if (!el) return false;
                    const s = getComputedStyle(el), r = el.getBoundingClientRect();
                    return s.display !== 'none' && s.visibility !== 'hidden' &&
                           r.width > 0 && r.height > 0;
                };

                const candidates = [...document.querySelectorAll('.inlineIcon.resource.transfer')]
                    .filter(visible);

                if (!candidates.length) {
                    return JSON.stringify({
                        ok:false,
                        error:'DOM .inlineIcon.resource.transfer tidak ditemukan'
                    });
                }

                const el = candidates.find(x =>
                    /openResourceTransfer/i.test(x.getAttribute('onclick') || '')
                ) || candidates[0];

                const onclick = el.getAttribute('onclick') || '';

                if (!/openResourceTransfer/i.test(onclick)) {
                    return JSON.stringify({
                        ok:false,
                        error:'onclick openResourceTransfer tidak ditemukan',
                        count:candidates.length
                    });
                }

                const getAmount = (name) => {
                    const re = new RegExp('\\b' + name + '\\s*:\\s*(\\d+)', 'i');
                    const m = onclick.match(re);
                    return m ? parseInt(m[1], 10) : 0;
                };

                const targetResourceAmount = {
                    lumber: getAmount('lumber'),
                    clay: getAmount('clay'),
                    iron: getAmount('iron'),
                    crop: getAmount('crop')
                };

                el.scrollIntoView({block:'center', inline:'center'});
                el.click();

                return JSON.stringify({
                    ok:true,
                    clickedClass: String(el.className || ''),
                    targetResourceAmount,
                    hasOpenResourceTransfer: /openResourceTransfer/i.test(onclick)
                });
            })()
        """.trimIndent()

        fun attempt(attempt: Int) {
            debugTrace("HERO TRANSFER: attempt $attempt/8")

            val targetWebView = automationWebView()

            if (targetWebView == null) {
                debugTrace("HERO TRANSFER: automationWebView() == null")

                if (attempt < 8) {
                    handler.postDelayed({ attempt(attempt + 1) }, 700L)
                } else {
                    logEvent(
                        "Resource Builder: WebView tidak tersedia; langsung mencoba upgrade"
                    )
                    clickResourceUpgrade()
                }
                return
            }

            targetWebView.evaluateJavascript(js) { result ->
                val decoded = result?.trim('"') ?: ""

                debugTrace(
                    "HERO TRANSFER: attempt $attempt/8 result=$decoded"
                )

                val normalized = decoded.replace("\\\"", "\"")
                if (decoded.contains("\"ok\":true") || normalized.contains("\"ok\":true")) {
                    debugTrace(
                        "HERO TRANSFER: BERHASIL klik .inlineIcon.resource.transfer"
                    )
                    handler.postDelayed({ clickTransferSelected() }, 900L)
                } else if (attempt < 8) {
                    debugTrace(
                        "HERO TRANSFER: resource transfer belum siap/gagal " +
                        "(attempt $attempt/8)"
                    )
                    handler.postDelayed({ attempt(attempt + 1) }, 700L)
                } else {
                    logEvent(
                        "Resource Builder: .inlineIcon.resource.transfer tidak ditemukan/gagal; langsung upgrade"
                    )
                    pendingUpgradeCosts = longArrayOf(0L, 0L, 0L, 0L)
                    handler.postDelayed({ clickResourceUpgrade() }, 300L)
                }
            }
        }

        attempt(1)
    }

private fun clickTransferSelected() {
        debugTrace("ENTER clickTransferSelected")
        if (!running || !builderInProgress || pendingUpgradeUrl.isBlank()) return

        val js = """
            (() => {
                const visible = el => {
                    if (!el) return false;
                    const s = getComputedStyle(el), r = el.getBoundingClientRect();
                    return s.display !== 'none' && s.visibility !== 'hidden' &&
                           s.opacity !== '0' && r.width > 0 && r.height > 0;
                };
                const norm = s => String(s || '').replace(/\s+/g,' ').trim().toLowerCase();

                // JANGAN pilih parent dialog (#reactDialogWrapper) yang kebetulan
                // mempunyai teks "Transfer resources". Cari kontrol tombol yang
                // benar-benar berisi "Transfer selected".
                const selectors = [
                    'button',
                    'input[type=button]',
                    'input[type=submit]',
                    '[role="button"]',
                    'a'
                ];

                let controls = [];
                for (const sel of selectors) {
                    try {
                        controls.push(...document.querySelectorAll(sel));
                    } catch (_) {}
                }

                controls = controls.filter(el =>
                    visible(el) &&
                    !el.disabled &&
                    el.getAttribute('aria-disabled') !== 'true'
                );

                const textOf = el => norm(
                    el.innerText || el.textContent || el.value ||
                    el.title || el.getAttribute('aria-label') || ''
                );

                // Prioritas 1: kontrol yang teksnya memang "Transfer selected"
                let btn = controls.find(el => {
                    const t = textOf(el);
                    return /transfer\s+selected/i.test(t);
                });

                // Prioritas 2: kontrol yang mempunyai kata Transfer + Selected
                // tetapi bukan ancestor besar yang hanya membungkus dialog.
                if (!btn) {
                    btn = controls.find(el => {
                        const t = textOf(el);
                        if (!/transfer/i.test(t) || !/selected/i.test(t)) return false;

                        const childMatch = [...el.querySelectorAll('button,a,[role="button"],input')]
                            .some(c => c !== el && visible(c) && /transfer/i.test(textOf(c)) && /selected/i.test(textOf(c)));
                        return !childMatch;
                    });
                }

                if (!btn) {
                    const dialog = document.querySelector('#reactDialogWrapper,[class*="reactDialog"]');
                    const dialogText = dialog ? norm(dialog.innerText || dialog.textContent || '') : '';
                    return JSON.stringify({
                        state:'not_found',
                        dialogVisible: !!dialog && visible(dialog),
                        dialogHasTransfer: /transfer/i.test(dialogText),
                        dialogText: dialogText.slice(0,500)
                    });
                }

                const beforeText = textOf(btn);
                const tag = btn.tagName;
                const cls = String(btn.className || '');
                btn.scrollIntoView({block:'center', inline:'center'});

                // Gunakan native click terlebih dahulu. React/Travian umumnya
                // menangani event ini lebih benar daripada mengklik wrapper.
                try { btn.click(); } catch (_) {
                    ['mousedown','mouseup','click'].forEach(type => {
                        try {
                            btn.dispatchEvent(new MouseEvent(type, {
                                bubbles:true, cancelable:true, view:window
                            }));
                        } catch (_) {}
                    });
                }

                return JSON.stringify({
                    state:'clicked',
                    tag,
                    text:beforeText.slice(0,200),
                    className:cls.slice(0,200)
                });
            })();
        """.trimIndent()

        automationWebView()?.evaluateJavascript(js) { raw ->
            val result = raw.orEmpty().trim('"').replace("\\\"", "\"")

            if (result.contains("\"state\":\"clicked\"")) {
                logEvent("Resource Builder: Transfer selected DIKLIK — menunggu popup memproses transfer")

                // Jangan langsung menganggap berhasil. Tunggu sebentar lalu
                // verifikasi apakah tombol Transfer Selected masih ada.
                handler.postDelayed({
                    verifyTransferSelectedCompleted()
                }, 1200L)
            } else if (inventoryUseAttempt < 15) {
                inventoryUseAttempt++
                debugTrace("HERO TRANSFER: tombol Transfer selected belum ditemukan, retry $inventoryUseAttempt/15")
                handler.postDelayed({ clickTransferSelected() }, 500L)
            } else {
                logEvent("Resource Builder: tombol Transfer selected tidak ditemukan setelah 15 percobaan")
                pendingUpgradeUrl = ""
                pendingUpgradeCosts = longArrayOf(0L,0L,0L,0L)
                goToNextBuilderVillage()
            }
        }
    }

    private fun verifyTransferSelectedCompleted() {
        debugTrace("ENTER verifyTransferSelectedCompleted")
        if (!running || !builderInProgress) return

        val js = """
            (() => {
                const visible = el => {
                    if (!el) return false;
                    const s=getComputedStyle(el), r=el.getBoundingClientRect();
                    return s.display!=='none' && s.visibility!=='hidden' &&
                           s.opacity!=='0' && r.width>0 && r.height>0;
                };
                const norm = s => String(s||'').replace(/\s+/g,' ').trim().toLowerCase();

                const controls = [
                    ...document.querySelectorAll('button'),
                    ...document.querySelectorAll('input[type=button],input[type=submit]'),
                    ...document.querySelectorAll('[role="button"]'),
                    ...document.querySelectorAll('a')
                ].filter(visible);

                const stillThere = controls.some(el => {
                    const t=norm(el.innerText||el.textContent||el.value||el.title||el.getAttribute('aria-label')||'');
                    return /transfer\s+selected/i.test(t);
                });

                const dialog = document.querySelector('#reactDialogWrapper,[class*="reactDialog"]');
                const dialogVisible = !!dialog && visible(dialog);

                return JSON.stringify({
                    stillThere,
                    dialogVisible,
                    dialogText: dialog ? norm(dialog.innerText||dialog.textContent||'').slice(0,300) : ''
                });
            })();
        """.trimIndent()

        automationWebView()?.evaluateJavascript(js) { raw ->
            val result = raw.orEmpty().trim('"').replace("\\\"", "\"")

            if (result.contains("\"stillThere\":false")) {
                logEvent("Resource Builder: Transfer Selected terkonfirmasi selesai — langsung Upgrade")
                heroTransferCompleted = true
                pendingUpgradeCosts = longArrayOf(0L,0L,0L,0L)
                handler.postDelayed({ clickResourceUpgrade() }, 700L)
            } else if (inventoryUseAttempt < 18) {
                inventoryUseAttempt++
                debugTrace("HERO TRANSFER: Transfer Selected masih ada; menunggu proses (${inventoryUseAttempt}/18)")
                handler.postDelayed({ verifyTransferSelectedCompleted() }, 700L)
            } else {
                logEvent("Resource Builder: Transfer Selected belum terkonfirmasi selesai; village dilewati demi mencegah upgrade palsu")
                pendingUpgradeUrl = ""
                pendingUpgradeCosts = longArrayOf(0L,0L,0L,0L)
                goToNextBuilderVillage()
            }
        }
    }

    private fun useHeroInventoryForPendingUpgrade(): Unit {
        debugTrace("ENTER useHeroInventoryForPendingUpgrade")
        if (!running || !builderInProgress || pendingUpgradeUrl.isBlank()) return
        inventoryUseAttempt++
        if (inventoryUseAttempt > 5) {
            logEvent("Resource Builder: gagal menggunakan resource Hero setelah 5 percobaan")
            pendingUpgradeUrl = ""
            goToNextBuilderVillage()
            return
        }

        val needed = pendingUpgradeCosts.joinToString(",")
        val js = """
            (() => {
                const needed = [$needed];
                const names = ['lumber','clay','iron','crop'];
                const patterns = [
                    /lumber|wood/i,
                    /clay/i,
                    /iron/i,
                    /crop/i
                ];
                const visible = el => {
                    if (!el) return false;
                    const s = getComputedStyle(el), r = el.getBoundingClientRect();
                    return s.display !== 'none' && s.visibility !== 'hidden' && r.width > 0 && r.height > 0;
                };
                const setValue = (el, value) => {
                    const setter = Object.getOwnPropertyDescriptor(Object.getPrototypeOf(el), 'value')?.set;
                    if (setter) setter.call(el, String(value)); else el.value = String(value);
                    el.dispatchEvent(new Event('input', {bubbles:true}));
                    el.dispatchEvent(new Event('change', {bubbles:true}));
                };
                const inventoryRoot = document.querySelector('#heroInventory, .heroInventory, .inventory, [class*="inventory"]') || document.body;
                const elements = [...inventoryRoot.querySelectorAll('*')].filter(visible);
                const metadata = el => [
                    el.getAttribute('title'), el.getAttribute('alt'), el.getAttribute('aria-label'),
                    el.getAttribute('data-item'), el.getAttribute('data-item-type'), el.getAttribute('data-type'),
                    (el.className || '').toString(), el.id || ''
                ].filter(Boolean).join(' ');

                let used = false;
                const missing = [];
                for (let i=0;i<4;i++) {
                    if (needed[i] <= 0) continue;
                    const candidate = elements.find(el => {
                        const meta = metadata(el);
                        return patterns[i].test(meta) && (
                            /item|resource|inventory|slot/i.test(meta) || el.tagName === 'IMG'
                        );
                    });
                    if (!candidate) { missing.push(names[i]); continue; }
                    const slot = candidate.closest('[data-item-id],[data-slot],.item,.slot,[class*="item"],[class*="slot"]') || candidate.parentElement || candidate;
                    slot.scrollIntoView({block:'center'});
                    candidate.click();
                    used = true;
                    break;
                }
                if (!used) return JSON.stringify({state:'no_item', missing});
                return JSON.stringify({state:'item_clicked'});
            })();
        """.trimIndent()
        automationWebView()?.evaluateJavascript(js) { raw ->
            val result = raw.orEmpty().trim('"').replace("\\\"", "\"")
            when {
                result.contains("no_item") -> {
                    logEvent("Resource Builder: item resource Hero tidak ditemukan untuk kebutuhan ${pendingUpgradeCosts.joinToString(",")}")
                    pendingUpgradeUrl = ""
                    goToNextBuilderVillage()
                }
                result.contains("item_clicked") -> {
                    handler.postDelayed({ fillHeroResourceDialog() }, 600)
                }
                else -> handler.postDelayed({ useHeroInventoryForPendingUpgrade() }, 700)
            }
        }
    }

    private fun fillHeroResourceDialog(): Unit {
        debugTrace("ENTER fillHeroResourceDialog")
        if (!running || !builderInProgress || pendingUpgradeUrl.isBlank()) return
        val needed = pendingUpgradeCosts.joinToString(",")
        val js = """
            (() => {
                const needed = [$needed];
                const names = ['lumber','clay','iron','crop'];
                const inputs = [
                    document.querySelector('input[name="lumber"]'),
                    document.querySelector('input[name="clay"]'),
                    document.querySelector('input[name="iron"]'),
                    document.querySelector('input[name="crop"]')
                ];
                const setValue = (el, value) => {
                    if (!el) return false;
                    const setter = Object.getOwnPropertyDescriptor(Object.getPrototypeOf(el), 'value')?.set;
                    if (setter) setter.call(el, String(value)); else el.value = String(value);
                    el.dispatchEvent(new Event('input', {bubbles:true}));
                    el.dispatchEvent(new Event('change', {bubbles:true}));
                    return true;
                };
                let found = 0;
                for (let i=0;i<4;i++) if (needed[i] > 0 && setValue(inputs[i], needed[i])) found++;
                if (!found) {
                    const all = [...document.querySelectorAll('input[type=number], input[type=text]')];
                    const candidates = all.filter(x => x.offsetParent !== null && !x.disabled);
                    for (let i=0;i<4 && i<candidates.length;i++) if (needed[i] > 0) { setValue(candidates[i], needed[i]); found++; }
                }
                if (!found) return 'no_amount_inputs';
                const buttons = [...document.querySelectorAll('button,a,input[type=submit],input[type=button],[role=button]')]
                    .filter(x => x.offsetParent !== null && !x.disabled);
                const norm = x => (x || '').replace(/\s+/g,' ').trim().toLowerCase();
                const confirm = buttons.find(x => /confirm|use|transfer|send|ok|done|accept/.test(norm(x.innerText || x.textContent || x.value || x.title || x.getAttribute('aria-label'))));
                if (!confirm) return 'no_confirm';
                confirm.click();
                return 'confirmed';
            })();
        """.trimIndent()
        automationWebView()?.evaluateJavascript(js) { raw ->
            val result = raw.orEmpty().trim('"').replace("\\\"", "\"")
            when {
                result == "confirmed" -> {
                    logEvent("Resource Builder: resource Hero digunakan (${pendingUpgradeCosts.joinToString(",")})")
                    handler.postDelayed({
                        automationWebView()?.loadUrl(pendingUpgradeUrl)
                    }, 900)
                }
                result == "no_amount_inputs" || result == "no_confirm" -> {
                    if (inventoryUseAttempt < 5) handler.postDelayed({ fillHeroResourceDialog() }, 800)
                    else {
                        logEvent("Resource Builder: dialog penggunaan resource Hero tidak dikenali")
                        pendingUpgradeUrl = ""
                        goToNextBuilderVillage()
                    }
                }
                else -> handler.postDelayed({ fillHeroResourceDialog() }, 800)
            }
        }
    }

    private fun clickResourceUpgrade() {
        debugTrace("ENTER clickResourceUpgrade")
        if (!running || !builderInProgress) return

        val currentUrl = automationWebView()?.url.orEmpty()
        if (!currentUrl.contains("build.php", ignoreCase = true) ||
            currentUrl.contains("gid=16", ignoreCase = true)) {
            logEvent("Resource Builder: batal klik Upgrade karena bukan halaman resource build.php; URL=$currentUrl")
            builderStage = "OPEN_RESOURCE"
            handler.postDelayed({ openSavedBuilderResource() }, 400)
            return
        }

        builderStage = "WAIT_UPGRADE"
        val js = """
            (() => {
                const visible = el => {
                    if (!el) return false;
                    const s = getComputedStyle(el), r = el.getBoundingClientRect();
                    return s.display !== 'none' && s.visibility !== 'hidden' && r.width > 0 && r.height > 0;
                };
                const norm = s => (s || '').replace(/\s+/g,' ').trim().toLowerCase();
                const root = document.querySelector('#build, #villageContent') || document.body;
                const all = [...root.querySelectorAll('button,a,input[type=submit],input[type=button],[role=button]')];
                const candidates = all.filter(el => visible(el) && !el.disabled && el.getAttribute('aria-disabled') !== 'true');
                const btn = candidates.find(el => {
                    const text = norm(el.innerText || el.textContent || el.value || el.title || el.getAttribute('aria-label'));
                    const cls = (el.className || '').toString().toLowerCase();
                    const href = (el.getAttribute('href') || '').toLowerCase();
                    if (/cancel|demolish|destroy|remove/.test(text + ' ' + cls)) return false;
                    return /upgrade|upgrade to level|build/.test(text) ||
                           /(?:^|\s)(green|build|upgrade)(?:\s|$)/.test(cls) ||
                           /build\.php/.test(href);
                }) || candidates.find(el => {
                    const cls = (el.className || '').toString().toLowerCase();
                    return /green/.test(cls) && /build|upgrade/.test(cls);
                });
                if (!btn) return 'not-found';
                btn.scrollIntoView({block:'center'});
                const href = btn.getAttribute('href') || '';
                if (href && /build\.php/i.test(href)) {
                    window.location.href = href;
                    return 'navigated:' + href;
                }
                btn.click();
                return 'clicked:' + (btn.innerText || btn.value || 'upgrade');
            })();
        """.trimIndent()
        automationWebView()?.evaluateJavascript(js) { raw ->
            val result = raw.orEmpty().trim('"').replace("\\\"", "\"")
            if (result.startsWith("clicked") || result.startsWith("navigated")) {
                pendingUpgradeUrl = ""
                pendingUpgradeCosts = longArrayOf(0L, 0L, 0L, 0L)
                heroTransferCompleted = false
                val villageLogName = builderVillages.getOrNull(builderVillageIndex)?.second ?: "Village ${builderVillageIndex + 1}"
                val villageLogId = builderVillages.getOrNull(builderVillageIndex)?.first.orEmpty()
                val currentLevel = builderResourceLevels[villageLogId] ?: -1
                val targetLevel = if (currentLevel >= 0) currentLevel + 1 else -1
                if (targetLevel >= 0) logEvent("Village $villageLogName Upgrade to Level $targetLevel Success")
                else logEvent("Village $villageLogName Upgrade Success")
                handler.postDelayed({ goToNextBuilderVillage() }, 1200)
            } else if (builderAttempt < 5) {
                builderAttempt++
                handler.postDelayed({ inspectUpgradeResources() }, 900)
            } else {
                pendingUpgradeUrl = ""
                val villageLogName = builderVillages.getOrNull(builderVillageIndex)?.second ?: "Village ${builderVillageIndex + 1}"
                logEvent("Village $villageLogName no upgrade")
                goToNextBuilderVillage()
            }
        }
    }

    private fun goToNextBuilderVillage() {
        debugTrace("ENTER goToNextBuilderVillage")
        if (!builderInProgress) return

        // Cegah callback ganda menaikkan index dua kali.
        if (builderStage == "ADVANCING") return
        builderStage = "ADVANCING"

        pendingBuilderResourceHref = ""
        builderVillageClickInProgress = false
        pendingUpgradeUrl = ""
        pendingUpgradeCosts = longArrayOf(0L, 0L, 0L, 0L)

        builderVillageIndex++
        handler.postDelayed({
            if (!running || !builderInProgress) return@postDelayed
            builderStage = "LOAD_DORF"
            processResourceBuilderVillage()
        }, 700)
    }

    private fun finishResourceBuilderCycle() {
        debugTrace("ENTER finishResourceBuilderCycle")
        val now = System.currentTimeMillis()
        if (resourceBuilderCycleStartedAt > 0L) {
            val duration = (now - resourceBuilderCycleStartedAt).coerceAtLeast(0L)
            getSharedPreferences(PREFS, MODE_PRIVATE).edit()
                .putLong("resource_cycle_duration_ms", duration)
                .putLong("resource_cycle_started_at", 0L)
                .apply()
            logEvent("Resource Builder: waktu proses ${formatDuration(duration)}")
            resourceBuilderCycleStartedAt = 0L
        }
        builderInProgress = false
        builderVillages.clear()
        builderVillageIndex = 0
        pendingBuilderResourceHref = ""
        builderVillageClickInProgress = false
        builderStage = "IDLE"
        builderStage = "IDLE"
        logEvent("CICLE END")
        scheduleNextRandomRun()
        updateNotification("Next Run ${timeFormat.format(Date(nextAt))} | dalam ${formatDuration((nextAt - System.currentTimeMillis()).coerceAtLeast(0L))}")
    }

    private fun loadBuilderVillagesFromSnapshot(): MutableList<Pair<String, String>> {
        debugTrace("ENTER loadBuilderVillagesFromSnapshot")
        val array = runCatching { org.json.JSONArray(selectedBuilderVillagesJson) }.getOrNull()
            ?: return mutableListOf()
        val out = mutableListOf<Pair<String, String>>()
        val seen = mutableSetOf<String>()
        for (i in 0 until array.length()) {
            val item = array.optJSONObject(i) ?: continue
            val id = item.optString("id").trim()
            val name = item.optString("name").trim().ifBlank { "Village $id" }
            if (id.isBlank() || !selectedBuilderVillageIds.contains(id) || !seen.add(id)) continue
            out.add(id to name)
        }
        // Jika selection tidak dikonfigurasi, gunakan semua village yang tersimpan
        // pada snapshot. Tidak ada discovery ulang dari sidebar.
        if (!builderSelectionConfigured && out.isEmpty()) {
            for (i in 0 until array.length()) {
                val item = array.optJSONObject(i) ?: continue
                val id = item.optString("id").trim()
                val name = item.optString("name").trim().ifBlank { "Village $id" }
                if (id.isNotBlank() && seen.add(id)) out.add(id to name)
            }
        }
        return out
    }

    private fun refreshBuilderSelectionFromPrefs() {
        debugTrace("ENTER refreshBuilderSelectionFromPrefs")
        val prefs = getSharedPreferences(PREFS, MODE_PRIVATE)
        builderSelectionConfigured = prefs.getBoolean("resource_builder_selection_configured", builderSelectionConfigured)
        selectedBuilderVillageIds = prefs.getStringSet(
            "resource_builder_selected_villages",
            selectedBuilderVillageIds
        )?.toSet() ?: emptySet()
        selectedBuilderVillagesJson = prefs.getString(
            "resource_builder_villages_json",
            selectedBuilderVillagesJson
        ).orEmpty()
        logEvent(
            "Resource Builder: selection terbaru dimuat — " +
                if (builderSelectionConfigured) {
                    if (selectedBuilderVillageIds.isEmpty()) "tidak ada village"
                    else selectedBuilderVillageIds.joinToString(", ")
                } else "SEMUA village"
        )
    }

    private fun absoluteBuilderHref(href: String): String {
        val clean = href.trim()
        if (clean.startsWith("http://", true) || clean.startsWith("https://", true)) return clean
        return when {
            clean.startsWith("/") -> server + clean
            clean.startsWith("./") -> "$server/${clean.removePrefix("./")}"
            else -> "$server/$clean"
        }
    }

    private fun discoverVillagesForBuilder() {
        debugTrace("ENTER discoverVillagesForBuilder")
        if (!running || !builderInProgress) return
        if (builderVillages.isNotEmpty() || builderStage != "DISCOVER" || builderDiscoverInFlight) return
        builderDiscoverInFlight = true
        val js = """
            (async () => {
                const villages = [];
                const seen = new Set();
                const clean = s => String(s || '').replace(/\s+/g,' ').trim();
                const add = (value, nameHint='') => {
                    const text = String(value || '');
                    const patterns = [
                        /[?&]newdid=(\d+)/i,
                        /(?:newdid|did|villageId|village_id)[=:'" ]+(\d+)/i
                    ];
                    let id = null;
                    if (/^\d+$/.test(text.trim())) {
                        // data-did contains the numeric ID directly.
                        id = text.trim();
                    } else {
                        for (const re of patterns) {
                            const m = text.match(re);
                            if (m) { id=m[1]; break; }
                        }
                    }
                    if (id && !seen.has(id)) {
                        seen.add(id);
                        villages.push({id, name: clean(nameHint) || ('Village ' + id)});
                    }
                };
                const scan = root => {
                    if (!root) return;

                    // Travian T4/T5: village entries are .listEntry and the
                    // village ID is stored in data-did. Do not depend only on
                    // href containing newdid because newer village-list markup
                    // can use data-did on the entry itself.
                    const entries = root.querySelectorAll('.listEntry');
                    for (const entry of entries) {
                        const id = entry.getAttribute('data-did') ||
                                   entry.dataset.did ||
                                   entry.getAttribute('data-village-id') ||
                                   entry.getAttribute('data-villageid');
                        const nameEl = entry.querySelector('.name');
                        const link = entry.querySelector('a[href]');
                        const name = clean(
                            nameEl ? (nameEl.textContent || '') :
                            (entry.getAttribute('data-name') || '')
                        );
                        if (id) add(id, name);
                        if (link) {
                            add(link.getAttribute('href'), name);
                            add(link.outerHTML, name);
                        }
                    }

                    // Older Travian markup / profile village list.
                    for (const a of root.querySelectorAll('a[href*="newdid="], a[title][href]')) {
                        const name = clean(
                            a.querySelector('.name')?.textContent ||
                            a.getAttribute('title') ||
                            a.textContent ||
                            a.getAttribute('aria-label') || ''
                        );
                        add(a.getAttribute('href'), name);
                    }

                    // Last-resort data attributes used by some layouts.
                    for (const el of root.querySelectorAll('[data-did],[data-village-id],[data-villageid],[data-newdid]')) {
                        const id = el.getAttribute('data-did') ||
                                   el.getAttribute('data-village-id') ||
                                   el.getAttribute('data-villageid') ||
                                   el.getAttribute('data-newdid');
                        const nameEl = el.querySelector?.('.name');
                        const name = clean(
                            nameEl ? nameEl.textContent :
                            el.getAttribute('title') ||
                            el.textContent || ''
                        );
                        add(id, name);
                    }
                };

                [document.querySelector('#sidebarBoxVillagelist'),
                 document.querySelector('#villageList'),
                 document.querySelector('#villageList .list'),
                 document.querySelector('#side_info'),
                 document.body].filter(Boolean).forEach(scan);

                const current = location.search.match(/[?&]newdid=(\d+)/i);
                if (current && !seen.has(current[1])) {
                    seen.add(current[1]);
                    villages.unshift({id: current[1], name: 'Village ' + current[1]});
                }

                const bodyText = (document.body.innerText || '').replace(/\s+/g, ' ');
                let expected = 0;
                const countMatch = bodyText.match(/VILLAGES\s+(\d+)\s*\/\s*\d+/i) ||
                                   bodyText.match(/VILLAGES\s*\(?\s*(\d+)\s*\/\s*\d+\)?/i);
                if (countMatch) expected = parseInt(countMatch[1], 10) || 0;

                if (expected > 0 && villages.length < expected) {
                    try {
                        let uid = null;
                        const profileLink = [...document.querySelectorAll('a[href]')]
                            .map(a => a.getAttribute('href') || '')
                            .find(h => /spieler\.php\?uid=\d+/i.test(h));
                        if (profileLink) {
                            const m = profileLink.match(/[?&]uid=(\d+)/i);
                            if (m) uid = m[1];
                        }
                        if (uid) {
                            const response = await fetch('spieler.php?uid=' + uid, {
                                credentials: 'include', cache: 'no-store'
                            });
                            const html = await response.text();
                            const doc = new DOMParser().parseFromString(html, 'text/html');
                            for (const root of [doc.querySelector('#villageList'),
                                                doc.querySelector('#sidebarBoxVillagelist'),
                                                doc.body].filter(Boolean)) scan(root);

                            // Profile pages have a dedicated village list. Parse each
                            // item explicitly so grouped/hidden sidebar villages are
                            // not lost.
                            for (const a of doc.querySelectorAll('#villageList .list li a[title][href], #villageList li a[href*="newdid="]')) {
                                const name = clean(a.getAttribute('title') || a.textContent || '');
                                add(a.getAttribute('href'), name);
                            }
                        }
                    } catch (e) {}
                }

                AndroidFarm.onVillageListResult(JSON.stringify({villages, expected}));
            })().catch(e => AndroidFarm.onVillageListResult(JSON.stringify({villages:[],expected:0,error:String(e)})));
        """.trimIndent()
        automationWebView()?.evaluateJavascript(js, null)
    }

    private fun handleVillageListResult(rawJson: String) {
        debugTrace("ENTER handleVillageListResult")
        builderDiscoverInFlight = false
        if (!running || !builderInProgress) return

        val json = runCatching { JSONObject(rawJson) }.getOrNull()
        val villages = mutableListOf<Pair<String, String>>()
        val villageArray = json?.optJSONArray("villages")
        if (villageArray != null) {
            for (i in 0 until villageArray.length()) {
                val item = villageArray.optJSONObject(i) ?: continue
                val id = item.optString("id").trim()
                val name = item.optString("name").trim().ifBlank { "Village $id" }
                if (id.isNotBlank()) villages.add(id to name)
            }
        }

        val uniqueVillages = villages.distinctBy { it.first }
        val expected = json?.optInt("expected", 0) ?: 0
        var selectedVillages = if (builderSelectionConfigured) {
            uniqueVillages.filter { selectedBuilderVillageIds.contains(it.first) }
        } else uniqueVillages

        // Gunakan daftar tersimpan dari UI bila halaman aktif tidak merender sidebar lengkap.
        if (builderSelectionConfigured && selectedVillages.size < selectedBuilderVillageIds.size) {
            val savedArray = runCatching { org.json.JSONArray(selectedBuilderVillagesJson) }.getOrNull()
            if (savedArray != null) {
                val savedVillages = mutableListOf<Pair<String, String>>()
                for (i in 0 until savedArray.length()) {
                    val item = savedArray.optJSONObject(i) ?: continue
                    val id = item.optString("id").trim()
                    val name = item.optString("name").trim().ifBlank { "Village $id" }
                    if (id.isNotBlank() && selectedBuilderVillageIds.contains(id)) {
                        savedVillages.add(id to name)
                    }
                }
                if (savedVillages.size >= selectedBuilderVillageIds.size) selectedVillages = savedVillages
            }
        }

        if (builderSelectionConfigured && selectedBuilderVillageIds.isEmpty()) {
            builderVillages = mutableListOf()
            logEvent("Resource Builder: tidak ada village yang dicentang; Builder dilewati pada siklus ini")
            finishResourceBuilderCycle()
            return
        }

        val completeSelectedList = builderSelectionConfigured &&
            selectedBuilderVillageIds.isNotEmpty() &&
            selectedVillages.size >= selectedBuilderVillageIds.size

        if (uniqueVillages.isEmpty() || ((expected > 0 && uniqueVillages.size < expected) && !completeSelectedList)) {
            if (builderAttempt < 10) {
                builderAttempt++
                logEvent("Resource Builder: village terdeteksi ${uniqueVillages.size}${if (expected > 0) "/$expected" else ""}; retry ${builderAttempt}/10")
                handler.postDelayed({ discoverVillagesForBuilder() }, 1800)
            } else {
                logEvent("Resource Builder: daftar village belum lengkap setelah 10 retry (${uniqueVillages.size}${if (expected > 0) "/$expected" else ""}); siklus dibatalkan agar tidak memproses sebagian village")
                finishResourceBuilderCycle()
            }
            return
        }

        builderVillages = selectedVillages.toMutableList()
        builderAttempt = 0
        builderStage = "LOAD_DORF"
        logEvent("Resource Builder: ${uniqueVillages.size} village ditemukan; ${selectedVillages.size} village dipilih: ${selectedVillages.joinToString(" | ") { "${it.second} [${it.first}]" }}")
        processResourceBuilderVillage()
    }

    private fun formatDuration(ms: Long): String {
        val totalSeconds = (ms.coerceAtLeast(0L) / 1000L)
        val hours = totalSeconds / 3600L
        val minutes = (totalSeconds % 3600L) / 60L
        val seconds = totalSeconds % 60L
        return String.format(Locale.getDefault(), "%02d:%02d:%02d", hours, minutes, seconds)
    }

    private fun persistActiveCycleDuration(now: Long = System.currentTimeMillis()) {
        val prefs = getSharedPreferences(PREFS, MODE_PRIVATE)
        val edit = prefs.edit()
        if (farmListCycleStartedAt > 0L) {
            edit.putLong("farm_cycle_duration_ms", (now - farmListCycleStartedAt).coerceAtLeast(0L))
            edit.putLong("farm_cycle_started_at", 0L)
            farmListCycleStartedAt = 0L
        }
        if (resourceBuilderCycleStartedAt > 0L) {
            edit.putLong("resource_cycle_duration_ms", (now - resourceBuilderCycleStartedAt).coerceAtLeast(0L))
            edit.putLong("resource_cycle_started_at", 0L)
            resourceBuilderCycleStartedAt = 0L
        }
        edit.apply()
    }

    private fun scheduleNextRandomRun() {
        debugTrace("ENTER scheduleNextRandomRun")
        if (!running) return
        persistActiveCycleDuration()
        handler.removeCallbacks(cycleWatchdogRunnable)
        val chosenMinutes = if (maxMinutes <= minMinutes) minMinutes
        else Random.nextLong(minMinutes, maxMinutes + 1)
        val delay = chosenMinutes * 60_000L
        val countdownStartedAt = System.currentTimeMillis()
        nextAt = countdownStartedAt + delay
        updateNextRun(delay)
        getSharedPreferences(PREFS, MODE_PRIVATE).edit()
            .putBoolean("cycle_active", false)
            .putLong("countdown_started_at", countdownStartedAt)
            .apply()
        // Selama countdown, Refresh Village belum selesai. Next Run akan menunggu
        // sampai refresh selesai/timeout sebelum memulai cycle.
        villageRefreshInProgress = false
        villageRefreshCompleted = false
        villageRefreshClosed = false
        scheduledRefreshForNextRun = true
        handler.removeCallbacks(nextRunRunnable)
        handler.postDelayed(nextRunRunnable, delay)
        scheduleVillageRefreshForNextRun(countdownStartedAt)
        logEvent("Next Run: ${timeFormat.format(Date(nextAt))}")
        updateNotification("Next Run ${timeFormat.format(Date(nextAt))} | dalam ${formatDuration(delay)}")
    }

    private val nextRunRunnable = Runnable {
        if (!running) return@Runnable
        logEvent("Countdown berakhir — memulai siklus")
        triggerScheduledCycle()
    }

    private fun updateNextRun(delayMs: Long) {
        debugTrace("ENTER updateNextRun")
        getSharedPreferences(PREFS, MODE_PRIVATE).edit()
            .putLong("next_run_at", if (delayMs == 0L) 0L else nextAt)
            .apply()
    }

    private fun acceptCookiesIfPresent(done: (String) -> Unit) {
        debugTrace("ENTER acceptCookiesIfPresent")
        val js = """
            (() => {
              const visible = el => {
                if (!el) return false; const s = getComputedStyle(el); const r = el.getBoundingClientRect();
                return s.display !== 'none' && s.visibility !== 'hidden' && r.width > 0 && r.height > 0;
              };
              const norm = s => (s || '').replace(/\s+/g, ' ').trim().toLowerCase();
              const roots = [document];
              for (let i=0;i<roots.length;i++) {
                let els=[]; try { els=[...roots[i].querySelectorAll('*')]; } catch(_) {}
                for (const el of els) if (el.shadowRoot && !roots.includes(el.shadowRoot)) roots.push(el.shadowRoot);
              }
              const selectors=['#cmpwelcomebtnyes a','#cmpwelcomebtnyes','.cmpboxbtnyes','#cmpbntyestxt','[class*="cmpboxbtnyes"]','[id*="cmpwelcomebtnyes"]'];
              let bannerVisible=false;
              for (const root of roots) {
                try { const box=root.querySelector('#cmpbox,#cmpbox2,.cmpbox,.cmpmore'); if(box&&visible(box)) bannerVisible=true; } catch(_){}
                for(const sel of selectors){ let el=null; try{el=root.querySelector(sel);}catch(_){} if(el&&visible(el)){try{el.click();return 'clicked';}catch(_){} } }
                let candidates=[]; try{candidates=[...root.querySelectorAll('button,a,input[type=button],input[type=submit],[role=button]')];}catch(_){}
                const accept=candidates.find(el=>visible(el)&&/^(accept all|accept all cookies|allow all|agree all|alle akzeptieren|tout accepter|aceptar todo)$/.test(norm(el.innerText||el.textContent||el.value||el.title||el.getAttribute('aria-label'))));
                if(accept){try{accept.click();return 'clicked';}catch(_){} }
              }
              const host=document.querySelector('#cmpwrapper'); if(host&&visible(host)) bannerVisible=true;
              return bannerVisible?'visible':'absent';
            })();
        """.trimIndent()
        automationWebView()?.evaluateJavascript(js) { raw -> done(raw.orEmpty().trim('"').lowercase(Locale.US)) }
    }

    @SuppressLint("SetJavaScriptEnabled")
    private fun recoverWebView() {
        debugTrace("ENTER recoverWebView")
        if (!running || webViewRecoveryInProgress) return
        webViewRecoveryInProgress = true
        try {
            webView?.stopLoading()
            webView?.destroy()
        } catch (_: Exception) {}
        webView = null
        ensureServiceWebView()
        webViewRecoveryInProgress = false

        val resumeUrl = when {
            loginInProgress || isLikelyLoginPage(lastAutomationUrl.lowercase(Locale.US)) -> server
            builderInProgress && lastAutomationUrl.contains("dorf1.php", ignoreCase = true) -> lastAutomationUrl
            pendingStartAll -> "$server/build.php?id=39&gid=16&tt=99"
            farmListEnabled -> "$server/build.php?id=39&gid=16&tt=99"
            else -> "$server/dorf1.php"
        }
        logEvent("RECOVERY: WebView baru siap; melanjutkan dari $resumeUrl")
        handler.postDelayed({
            if (running) automationWebView()?.loadUrl(resumeUrl)
        }, 250L)
    }

    private fun stopAutomation(stopStartId: Int? = null) {
        debugTrace("ENTER stopAutomation")
        persistActiveCycleDuration()
        // Nonaktifkan bot = hentikan siklus yang sedang berjalan dan seluruh callback tertunda.
        running = false
        builderInProgress = false
        farmListCycleComplete = false
        countdownCyclePending = false
        pendingStartAll = false
        handler.removeCallbacks(cycleWatchdogRunnable)
        handler.removeCallbacks(delayedVillageRefreshRunnable)
        villageRefreshInProgress = false
        villageRefreshCompleted = false
        villageRefreshInspectInFlight = false
        villageRefreshVillages.clear()
        pendingStartAll = false
        handler.removeCallbacksAndMessages(null)
        if (webView != null) {
            webView?.destroy()
            webView = null
        }
        getSharedPreferences(PREFS, MODE_PRIVATE).edit()
            .putBoolean("service_running", false)
            .putLong("next_run_at", 0L)
            .putBoolean("cycle_active", false)
            .putBoolean("farm_list_enabled", farmListEnabled)
            .putBoolean("resource_builder_enabled", resourceBuilderEnabled)
            .apply()
        logEvent("Background service dihentikan")
        stopForeground(STOP_FOREGROUND_REMOVE)
        // Hanya hentikan service jika request STOP ini masih merupakan startId terbaru.
        // Jika user cepat menekan OFF lalu ON, ACTION_START mendapat startId baru
        // sehingga service lama tidak ikut mematikan instance yang baru aktif.
        if (stopStartId != null) {
            stopSelfResult(stopStartId)
        } else {
            stopSelf()
        }
    }

    private fun updateNotification(text: String) {
        debugTrace("ENTER updateNotification")
        val manager = getSystemService(NotificationManager::class.java)
        manager.notify(NOTIFICATION_ID, buildNotification(text))
    }

    private fun buildNotification(text: String): Notification {
        debugTrace("ENTER buildNotification")
        val intent = Intent(this, MainActivity::class.java)
        val pending = PendingIntent.getActivity(
            this, 0, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            Notification.Builder(this, CHANNEL_ID)
                .setContentTitle("Travian Farm Assistant — AKTIF")
                .setContentText(text)
                .setSmallIcon(android.R.drawable.ic_popup_sync)
                .setOngoing(true)
                .setCategory(Notification.CATEGORY_SERVICE)
                .setOnlyAlertOnce(true)
                .setShowWhen(false)
                .setContentIntent(pending)
                .build()
        } else {
            @Suppress("DEPRECATION")
            Notification.Builder(this)
                .setContentTitle("Travian Farm Assistant — AKTIF")
                .setContentText(text)
                .setSmallIcon(android.R.drawable.ic_popup_sync)
                .setOngoing(true)
                .setCategory(Notification.CATEGORY_SERVICE)
                .setOnlyAlertOnce(true)
                .setShowWhen(false)
                .setContentIntent(pending)
                .build()
        }
    }

    private fun createNotificationChannel() {
        debugTrace("ENTER createNotificationChannel")
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            getSystemService(NotificationManager::class.java).createNotificationChannel(
                NotificationChannel(CHANNEL_ID, "Farm Assistant Background", NotificationManager.IMPORTANCE_LOW)
            )
        }
    }

    private fun normalizeServer(value: String): String {
        debugTrace("ENTER normalizeServer")
        var s = value.trim()
        if (s.isBlank()) s = "https://ts20.x2.europe.travian.com"
        if (!s.startsWith("http")) s = "https://$s"
        return s.trimEnd('/')
    }

    /** Debug tracing dinonaktifkan untuk build produksi. */
    private fun debugTrace(message: String) {
        // Intentionally empty.
    }

    private fun logEvent(message: String) {
        val clean = when {
            message == "CICLE START" -> "CICLE START"
            message == "Click Send All Farmlist Success" -> "Click Send All Farmlist Success"
            message == "CICLE END" -> "CICLE END"
            message.startsWith("Village ") && message.contains(" Upgrade to Level ") && message.endsWith(" Success") -> message
            message.startsWith("Village ") && message.endsWith(" Upgrade Success") -> message
            message.startsWith("Village ") && message.endsWith(" no upgrade") -> message
            message.startsWith("Village ") && message.contains(" Updated min L") -> message
            message.startsWith("Next Run: ") -> message
            message == "REFRESH VILLAGE START" -> "REFRESH VILLAGE START"
            message == "REFRESH VILLAGE END" -> "REFRESH VILLAGE END"
            message == "BOT ON" -> "BOT ON"
            message == "BOT OFF" -> "BOT OFF"
            else -> return
        }
        val line = "${logTimeFormat.format(Date())} | $clean"
        try {
            openFileOutput(logFileName, MODE_APPEND).bufferedWriter().use { it.appendLine(line) }
        } catch (_: Exception) {
            // Logging must never interrupt the automation.
        }
    }

    private fun pruneLogs() {
        debugTrace("ENTER pruneLogs")
        try {
            val file = getFileStreamPath(logFileName)
            if (!file.exists()) return
            val cutoff = System.currentTimeMillis() - logMaxAgeMs
            val kept = file.readLines().filter { line ->
                try { logTimeFormat.parse(line.substringBefore(" | "))?.time ?: 0L >= cutoff }
                catch (_: Exception) { false }
            }
            file.writeText(kept.joinToString("\n") + if (kept.isNotEmpty()) "\n" else "")
        } catch (_: Exception) {}
    }

    inner class FarmBridge {
        @JavascriptInterface
        fun onLoginResult(result: String) {
            debugTrace("ENTER onLoginResult")
            handler.post {
                if (!running) return@post
                when (result) {
                    "submitting" -> updateNotification("Farm Assistant — mengirim login")
                    "no_login_form" -> {
                        loginInProgress = false
                        reloginRequested = false
                        loginRetryCount = 0
                        logEvent("Session aktif terdeteksi; membuka Farm List")
                        handler.postDelayed({ triggerStartAllFarmLists() }, 250)
                    }
                    "no_username_field", "no_form" -> {
                        if (loginRetryCount < 20 && running) handler.postDelayed({ autoLoginIfNeeded() }, 1000)
                        else {
                            loginInProgress = false
                            reloginRequested = false
                            logEvent("Form login Travian tidak dikenali")
                        }
                    }
                }
            }
        }

        @JavascriptInterface
        fun onVillageListResult(result: String) {
            debugTrace("ENTER onVillageListResult")
            handler.post {
                handleVillageListResult(result)
            }
        }
    }

    override fun onDestroy() {
        debugTrace("ENTER onDestroy")
        handler.removeCallbacksAndMessages(null)
        webView?.destroy()
        webView = null
        instanceRef = null
        visibleWebViewRef = null
        super.onDestroy()
    }

    override fun onBind(intent: Intent?): IBinder? {
        debugTrace("ENTER onBind")
        return null
    }
}
