/// <reference path="../../VerifyInfoPage.aspx" />
var NumberLoginAttempts;
var Login = Backbone.View.extend({
    id: "Login",
    lastScrollPos: 0,
    displayError: "",
    events: {
        'click #registerBtn': 'Login_click',
        'click #liForgotPassword': 'liForgotPassword_click',
        'click #liLoginWithOtp': 'liLoginWithOtp_click',
        'click #liSocialMedia': 'liSocialMedia_click',
        'click #aCreateClinic': 'aCreateClinic_click',
        'mousedown #eye-btn': 'showPassword',
        'mouseup #eye-btn': 'hidePassword'
    },
    initialize: function (option) {
        this.cid = "Login";
        this.options = option;
    },
    render: function () {
        !$("body").removeClass("logged-in")
        this.$el.html(this.template());
        var navigateAfter = getParameterByName("NavigateTo");
        if ((this.options && this.options.registerOnlyUser) || (navigateAfter != null && navigateAfter != "")) {
            if (this.options && this.options.userEmail && this.options.userEmail != "") {
                $("#txtUserName").val(this.options.userEmail);
            }
            if (navigateAfter == "PatientQuestionnaire" || (this.options && this.options.navigateAfter == "PatientQuestionnaire")) {
                $("#aCreateClinic").hide();
            }
        }
        isMoveToPageInprogress = false;
        translate()
        getCaptchaLang().then(function (langKey) {
            var JSLink = "https://www.google.com/recaptcha/api.js?hl=" + langKey;
            var JSElement = document.createElement('script');
            JSElement.src = JSLink;
            JSElement.id = "reCaptchaScript";
            document.getElementsByTagName('head')[0].appendChild(JSElement);
            if (checkIfMobile()) {
                $("#recaptcha").attr("data-size", "compact");
                checkAndRelocationRecaptchaPopupChallenge();
            }
        });

    },
    Login_click: function (event) {
        $(".outInputError").hide();
        if ($("#txtUserName").val() == "") {
            $("#txtUserName").parent("Div").addClass("inputErrorBorderColor");
            $(".txtUserName.inputErrorText").css({ "max-height": "50px" });
            return;
        }
        if ($('#txtPassword').val() == "") {
            $("#txtPassword").parent("Div").addClass("inputErrorBorderColor");
            $(".txtPassword.inputErrorText").css({ "max-height": "50px" });
            return;
        }
        $captcha = $('#recaptcha');
        if (($captcha).length > 0) {
            response = grecaptcha.getResponse();

            if (response.length === 0) {
                $(".kontakt_form.inputErrorText").css({ "max-height": "50px" });
                $(".kontakt_form.inputErrorText").show();
                if (!$captcha.hasClass("error")) {
                    $captcha.addClass("error");
                    return;
                }
                return;
            }
        }
        var base = this;
        var postData = { UserName: $("#txtUserName").val(), Password: $("#txtPassword").val() };
        LoginRegisterServerApi.LoginUser(postData).then(function (response) {
            if (response.Success == false && response.Message != null) {
                $(".outInputError").show();
                setTextTranslated(".inputError", response.Message);
                logError("User failed to login", { error: response.Message });
                //$(".inputError").html(response.Message);
                return;
                //  }
            }
            if (!response.Success && !response.PassedThroteling) {
                logError("User failed to login - wrong username and password", { UserName: $("#txtUserName").val() });
                alert(getTranslatedText("html-async-message-general-error"));
                return;
            }
            if (response.Success == false) {
                //alert("שם משתמש או סיסמא שגויים");
                logError("User failed to login - wrong username and password", { UserName: $("#txtUserName").val() });
                alert(getTranslatedText("invalid-password-or-user-name"));
                return;
            }
            if (base.options && base.options.registerOnlyUser) {
                if (base.options.navigateAfter == "LiveVideoTherapist") {
                    window.location.replace("/AnonymousPages/PrivateClinics/LoginPage.aspx#LiveVideoTherapist?verificationData=6203A105-1F7F-47F3-977E-7F9193BB7C51")
                    return;
                }
                else {
                    router.navigate(base.options.navigateAfter, true);
                    return;
                }
            }
            var navigateAfter = getParameterByName("NavigateTo");
            if (navigateAfter != null && navigateAfter != "") {//after recover password
                if (navigateAfter != "LiveVideoTherapist") {
                    router.navigate(navigateAfter, true);
                    return;
                }
                else {
                    window.location.replace("/AnonymousPages/PrivateClinics/LoginPage.aspx#LiveVideoTherapist?verificationData=6203A105-1F7F-47F3-977E-7F9193BB7C51")
                    return;
                }
            }
            switch (response.ResultObj) {
                case "1":
                case "2":
                    window.location.href = "VerifyInfoPage.aspx?Step=" + response.ResultObj;
                    break;
                default: {
                    var expUtils = new ExperimentsUtils();
                    expUtils.conductingExperiment(ExperimentKeys.LCHideSomeUnNecessaryPagesForTherapistFT, "false", base.afterExperimentHideSomeUnNecessaryPages, "", response.ResultObj, null, getOrganization());
                }
            }

        });
    },
    afterExperimentHideSomeUnNecessaryPages: function (response, loginStep) {
        if (response == 'true') {
            delete sessionStorage["Step"]
            window.location.replace("/SecurityInfrastructure/PrivateClinics/DashboardPage.aspx")
        }
        else {
            switch (loginStep) {
                case "3":
                case "4":
                    window.location.href = "VerifyInfoPage.aspx?Step=" + response.ResultObj;
                case "5":
                    window.location.replace("/SecurityInfrastructure/PrivateClinics/DashboardPage.aspx")
                    break;

            }
        }
    },
    liForgotPassword_click: function (event) {
        var option = this.options && this.options.navigateAfter ? this.options.navigateAfter : "";
        moveToNewView(RecoverPassword, { navigateAfter: option }, fade)
    },
    liLoginWithOtp_click: function (event) {
    },
    liSocialMedia_click: function (event) {
    },
    aCreateClinic_click: function (event) {
        if (this.options && this.options.registerOnlyUser) {
            router.navigate('Register/' + this.options.navigateAfter, true);
            return;
        }
        else {
            router.navigate('CreateClinic', true);
        }
    },
    showPassword: function () {
        $("#eye-btn").css("background-image", "url(pic/closeEye.png)");
        $('#txtPassword').attr("type","text");
    },
    hidePassword: function () {
        $("#eye-btn").css("background-image", "url(pic/openEye.png)");
        $('#txtPassword').attr("type", "password");
    }
});
function vcc(g_recaptcha_response) {
    var $captcha = $('#recaptcha');
    $('.kontakt_form').text('');
    $('.kontakt_form').hide();
    $captcha.removeClass("error");
    validItem("kontakt_form");
};
function checkAndRelocationRecaptchaPopupChallenge() {
    setInterval(function () {
        let $reCaptchaIframe = $('iframe[title="recaptcha challenge"]');
        if ($reCaptchaIframe.length > 0) {
            let $reCaptchaOverlay = $reCaptchaIframe.parent().parent();
            var centerWidth = (window.innerWidth - $reCaptchaOverlay.width()) / 2;
            $reCaptchaOverlay.css("left", centerWidth);
            $(".g-recaptcha-bubble-arrow").hide();
        }
    }, 100);
}
