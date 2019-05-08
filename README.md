## To execute
mvn clean install
mvn jetty:run

Then, go to the app -> http://127.0.0.1:8080/

> **NOTE:** This branch (`outlook-api`) is a snapshot of the tutorial before it was updated to use the [Microsoft Graph API](https://developer.microsoft.com/en-us/graph/) via the [Microsoft Graph SDK for Python](https://github.com/microsoftgraph/msgraph-sdk-python). Microsoft recommends using the Microsoft Graph to access Outlook mail, calendar, and contacts. You should use the Outlook APIs directly (via https://outlook.office.com/api) only if you require a feature that is not available on the Graph endpoints. For the Graph version of this tutorial, see the `master` branch.

## Before you begin

You need to install the [Java SE Development Kit (JDK)](http://www.oracle.com/technetwork/java/javase/downloads/index-jsp-138363.html#javasejdk). This guide was written with JDK 8 Update 92.

## Designing the app ##

Our app will be very simple. When a user visits the site, they will see a button to log in and view their email. Clicking that button will take them to the Azure login page where they can login with their Office 365 or Outlook.com account and grant access to our app. Finally, they will be redirected back to our app, which will display a list of the most recent email in the user's inbox.


## Implementing OAuth2 ##

Our goal in this section is to make the link on our home page initiate the [OAuth2 Authorization Code Grant flow with Azure AD](https://msdn.microsoft.com/en-us/library/azure/dn645542.aspx).

Before we proceed, we need to register our app to obtain a client ID and secret. Head over to https://apps.dev.microsoft.com to quickly get a client ID and secret. Using the sign in buttons, sign in with either your Microsoft account (Outlook.com), or your work or school account (Office 365).

Once you're signed in, click the **Add an app** button. Enter `java-tutorial` for the name and click **Create application**. After the app is created, locate the **Application Secrets** section, and click the **Generate New Password** button. Copy the password now and save it to a safe place. Once you've copied the password, click **Ok**.

![The new password dialog.](./readme-images/new-password.PNG)

Locate the **Platforms** section, and click **Add Platform**. Choose **Web**, then enter `http://localhost:8080/authorize.html` under **Redirect URIs**. Click **Save** to complete the registration. Copy the **Application Id** and save it along with the password you copied earlier. We'll need those values soon.

Here's what the details of your app registration should look like when you are done.

![The completed registration properties.](./readme-images/app-registration.PNG)

In **Project Explorer**, expand **Java Resources**. Right-click **src/main/resources** and choose **New**, then **Other**. Expand **General** and choose **File**. Name the file `auth.properties` and click **Finish**. Add the following lines to the file, replacing `YOUR_APP_ID_HERE` with your application ID, and `YOUR_APP_PASSWORD_HERE` with your application password.

```INI
appId=YOUR_APP_ID_HERE
appPassword=YOUR_APP_PASSWORD_HERE
redirectUrl=http://localhost:8080/authorize.html
```
### Exchanging the code for a token ###

Save all of your changes, restart the app, and browse to http://localhost:8080. This time if you log in, you should see an access token. 

### Refreshing the access token

Access tokens returned from Azure are valid for an hour. If you use the token after it has expired, the API calls will return 401 errors. You could ask the user to sign in again, but the better option is to refresh the token silently.

Now if you save all of your changes, restart the app, then login, you should end up on a rather empty-looking mail page. If you check the **Console** window in Spring Tool Suites, you should be able to verify that the API call worked by looking for the Retrofit logging entries. You should see something like this:

```
--> GET https://outlook.office.com/api/v2.0/me/mailfolders/inbox/messages?$orderby=ReceivedDateTime%20DESC&$select=ReceivedDateTime,From,IsRead,Subject,BodyPreview&$top=10 http/1.1
User-Agent: java-tutorial
client-request-id: 3d42cd86-f74b-40d9-9dd3-031de58fec0f
return-client-request-id: true
X-AnchorMailbox: AllieB@contoso.com
Authorization: Bearer eyJ0eXAiOiJK...
--> END GET
```

That should be followed by a `200 OK` line. If you scroll past the response headers, you should find a response body.

### For Calendar API: ###

1. Update the `scopes` array in `AuthHelper.java` to include the `Calendars.Read` scope.
  
  ```java
  private static String[] scopes = { 
    "openid", 
    "offline_access",
    "profile", 
    "email", 
    "https://outlook.office.com/mail.read",
    "https://outlook.office.com/calendars.read"
};
  ```

### For Contacts API: ###

1. Update the `scopes` array in `AuthHelper.java` to include the `Contacts.Read` scope.

  ```java
  private static String[] scopes = { 
    "openid", 
    "offline_access",
    "profile", 
    "email", 
    "https://outlook.office.com/mail.read",
    "https://outlook.office.com/calendars.read",
    "https://outlook.office.com/contacts.read"
};
  ```
1. Create a class in the `com.outlook.dev.service` package for the [Contact entity](https://msdn.microsoft.com/office/office365/api/complex-types-for-mail-contacts-calendar#RESTAPIResourcesContact).
  ```java
  package com.outlook.dev.service;

  import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
  import com.fasterxml.jackson.annotation.JsonProperty;

  @JsonIgnoreProperties(ignoreUnknown = true)
  public class Contact {
    @JsonProperty("Id")
    private String id;
    @JsonProperty("GivenName")
    private String givenName;
    @JsonProperty("Surname")
    private String surname;
    @JsonProperty("CompanyName")
    private String companyName;
    @JsonProperty("EmailAddresses")
    private EmailAddress[] emailAddresses;
    
    public String getId() {
      return id;
    }
    public void setId(String id) {
      this.id = id;
    }
    public String getGivenName() {
      return givenName;
    }
    public void setGivenName(String givenName) {
      this.givenName = givenName;
    }
    public String getSurname() {
      return surname;
    }
    public void setSurname(String surname) {
      this.surname = surname;
    }
    public String getCompanyName() {
      return companyName;
    }
    public void setCompanyName(String companyName) {
      this.companyName = companyName;
    }
    public EmailAddress[] getEmailAddresses() {
      return emailAddresses;
    }
    public void setEmailAddresses(EmailAddress[] emailAddresses) {
      this.emailAddresses = emailAddresses;
    }
  }
  ```
1. Add a `getContacts` function to the the `OutlookService` interface.

  ```java
  @GET("/api/v2.0/me/contacts")
Call<PagedResult<Contact>> getContacts(
      @Query("$orderby") String orderBy,
      @Query("$select") String select,
      @Query("$top") Integer maxResults
);
  ```
1. Add a controller for viewing contacts to the `com.outlook.dev.controller` package.
  ```java
  package com.outlook.dev.controller;

  import java.io.IOException;
  import java.util.Date;

  import javax.servlet.http.HttpServletRequest;
  import javax.servlet.http.HttpSession;

  import org.springframework.stereotype.Controller;
  import org.springframework.ui.Model;
  import org.springframework.web.bind.annotation.RequestMapping;
  import org.springframework.web.servlet.mvc.support.RedirectAttributes;

  import com.outlook.dev.auth.AuthHelper;
  import com.outlook.dev.auth.TokenResponse;
  import com.outlook.dev.service.Contact;
  import com.outlook.dev.service.OutlookService;
  import com.outlook.dev.service.OutlookServiceBuilder;
  import com.outlook.dev.service.PagedResult;

  @Controller
  public class ContactsController {
    @RequestMapping("/contacts")
    public String contacts(Model model, HttpServletRequest request, RedirectAttributes redirectAttributes) {
      HttpSession session = request.getSession();
      TokenResponse tokens = (TokenResponse)session.getAttribute("tokens");
      if (tokens == null) {
        // No tokens in session, user needs to sign in
        redirectAttributes.addFlashAttribute("error", "Please sign in to continue.");
        return "redirect:/index.html";
      }
      
      String tenantId = (String)session.getAttribute("userTenantId");
		
		  tokens = AuthHelper.ensureTokens(tokens, tenantId);
      
      String email = (String)session.getAttribute("userEmail");
      
      OutlookService outlookService = OutlookServiceBuilder.getOutlookService(tokens.getAccessToken(), email);
      
      // Sort by given name in ascending order (A-Z)
      String sort = "GivenName ASC";
      // Only return the properties we care about
      String properties = "GivenName,Surname,CompanyName,EmailAddresses";
      // Return at most 10 contacts
      Integer maxResults = 10;
      
      try {
        PagedResult<Contact> contacts = outlookService.getContacts(
            sort, properties, maxResults)
            .execute().body();
        model.addAttribute("contacts", contacts.getValue());
      } catch (IOException e) {
        redirectAttributes.addFlashAttribute("error", e.getMessage());
        return "redirect:/index.html";
      }
      
      return "contacts";
    }
  }
  ```
1. Add `contacts.jsp` in the **jsp** folder.
  ```jsp
  <%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
  <%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>

  <c:if test="${error ne null}">
    <div class="alert alert-danger">Error: ${error}</div>
  </c:if>

  <table class="table">
    <caption>Contacts</caption>
    <thead>
      <tr>
        <th>Name</th>
        <th>Company</th>
        <th>Email</th>
      </tr>
    </thead>
    <tbody>
      <c:forEach items="${contacts}" var="contact">
        <tr>
          <td><c:out value="${contact.givenName} ${contact.surname}" /></td>
          <td><c:out value="${contact.companyName}" /></td>
          <td>
            <ul class="list-inline">
              <c:forEach items="${contact.emailAddresses}" var="address">
                <li><c:out value="${address.address}" /></li>
              </c:forEach>
            </ul>
          </td>
        </tr>
      </c:forEach>
    </tbody>
  </table>
  ```
1. Add a page definition for `events.jsp` in `pages.xml`.

  ```xml
  <definition name="contacts" extends="common">
    <put-attribute name="title" value="My Contacts" />
    <put-attribute name="body" value="/WEB-INF/jsp/contacts.jsp" />
    <put-attribute name="current" value="contacts" />
  </definition>
  ```
1. Add a nav bar entry for the events view in `base.jsp`.

  ```jsp
  <li class="${current == 'contacts' ? 'active' : '' }">
    <a href="<spring:url value="/contacts.html" />">Contacts</a>
  </li>
  ```
1. Restart the app.

## Next Steps ##

Now that you've created a working sample, you may want to learn more about the [capabilities of the Mail API](https://msdn.microsoft.com/office/office365/APi/mail-rest-operations). If your sample isn't working, and you want to compare, you can download the end result of this tutorial from [GitHub](https://github.com/jasonjoh/java-tutorial). If you download the project from GitHub, be sure to put your application ID and secret into the code before trying it.

## Copyright ##

Copyright (c) Microsoft. All rights reserved.

----------
Connect with me on Twitter [@JasonJohMSFT](https://twitter.com/JasonJohMSFT)

Follow the [Outlook/Exchange Dev Blog](https://blogs.msdn.microsoft.com/exchangedev/)
