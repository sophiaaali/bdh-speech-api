# Detailed Design: Add MFA Message in Account Menu

## Overview

This document outlines the detailed design for adding an MFA alert message to the AWS Console's Account Menu. The feature aims to improve account security by prompting users who don't have MFA enabled to set it up. The alert will be displayed in the Account Menu dropdown and will be dismissible, with the dismissed state persisted in localStorage.

## Requirements

Based on the requirements clarification process, the feature must:

1. Show an MFA alert for both IAM users and root users who don't have MFA enabled
2. Use the IAM:listMFADevices API to determine if MFA is enabled
3. Use CloudScape's Alert component with a dismissible property
4. Persist the dismissed state in localStorage (with future migration to CCS)
5. Construct the MFA setup URL using the current region
6. Implement performance optimizations to minimize impact on page load
7. Support localization using the existing Nav localization setup
8. Implement metrics for tracking user interactions
9. Be deployed behind a feature flag using FMS

## Architecture

### Component Structure

The feature will be implemented as a new React component that will be integrated into the existing Account Menu component in the AWS Console Navigation.

```
AccountMenu/
├── AccountMenuDropdown/
│   ├── UserInfo/
│   ├── MFAAlert/ (new component)
│   └── MenuItems/
└── ...
```

### Data Flow

1. When the Account Menu is opened, the MFAAlert component will check:
   - If the feature flag is enabled
   - If the user is an IAM user or root user (not a role)
   - If the user has MFA enabled (via IAM:listMFADevices API)
   - If the user has previously dismissed the alert (via localStorage)

2. If all conditions are met (feature enabled, IAM/root user, no MFA, not dismissed), the alert will be displayed.

3. When the user clicks "Setup MFA", they will be redirected to the IAM security credentials page.

4. When the user dismisses the alert, the preference will be stored in localStorage.

## Components and Interfaces

### MFAAlert Component

```jsx
import React, { useState, useEffect } from 'react';
import { Alert, Button } from '@cloudscape-design/components';
import { useTranslation } from 'react-i18next';
import { useFeatureFlag } from '@aws-console/feature-flags';
import { useRegion } from '@aws-console/page-context';
import { useMetrics } from '@aws-console/metrics';

const MFA_ALERT_DISMISSED_KEY = 'aws-console-mfa-alert-dismissed';

export const MFAAlert = ({ userType }) => {
  const [alertVisible, setAlertVisible] = useState(false);
  const [mfaEnabled, setMfaEnabled] = useState(null);
  const { t } = useTranslation();
  const { isEnabled } = useFeatureFlag();
  const { region } = useRegion();
  const metrics = useMetrics();
  
  // Check if feature flag is enabled
  const isMfaAlertEnabled = isEnabled('mfa-alert');
  
  // Check if user has MFA enabled
  useEffect(() => {
    if (!isMfaAlertEnabled || !['IAM', 'ROOT'].includes(userType)) {
      return;
    }
    
    const checkMfaStatus = async () => {
      // Check cache first
      const cachedStatus = localStorage.getItem('mfaEnabled');
      const cacheTimestamp = localStorage.getItem('mfaEnabledTimestamp');
      
      if (cachedStatus && cacheTimestamp) {
        const cacheAge = Date.now() - parseInt(cacheTimestamp, 10);
        if (cacheAge < 24 * 60 * 60 * 1000) { // Less than 1 day
          setMfaEnabled(cachedStatus === 'true');
          return;
        }
      }
      
      try {
        const startTime = Date.now();
        const response = await AWS.IAM.listMFADevices({});
        const endTime = Date.now();
        
        // Record API latency metric
        metrics.recordLatency('MFA_API_LATENCY', endTime - startTime);
        
        const hasMfa = response.MFADevices.length > 0;
        setMfaEnabled(hasMfa);
        
        // Update cache
        localStorage.setItem('mfaEnabled', hasMfa.toString());
        localStorage.setItem('mfaEnabledTimestamp', Date.now().toString());
      } catch (error) {
        console.error('Error checking MFA status:', error);
        // Default to not showing the alert on error
        setMfaEnabled(true);
      }
    };
    
    checkMfaStatus();
  }, [isMfaAlertEnabled, userType, metrics]);
  
  // Check if alert should be visible
  useEffect(() => {
    if (mfaEnabled === null) {
      return; // Still loading
    }
    
    const isDismissed = localStorage.getItem(MFA_ALERT_DISMISSED_KEY) === 'true';
    const shouldShowAlert = isMfaAlertEnabled && !mfaEnabled && !isDismissed;
    
    setAlertVisible(shouldShowAlert);
    
    if (shouldShowAlert) {
      // Record impression metric
      metrics.recordCount('MFA_ALERT_IMPRESSION');
    }
  }, [isMfaAlertEnabled, mfaEnabled, metrics]);
  
  // Handle Setup MFA button click
  const handleSetupMFA = () => {
    metrics.recordCount('MFA_SETUP_BUTTON_CLICK');
    const mfaSetupUrl = `https://${region}.console.aws.amazon.com/iam/home?region=${region}#/security_credentials`;
    window.location.href = mfaSetupUrl;
  };
  
  // Handle alert dismiss
  const handleDismiss = () => {
    metrics.recordCount('MFA_ALERT_DISMISS');
    localStorage.setItem(MFA_ALERT_DISMISSED_KEY, 'true');
    setAlertVisible(false);
  };
  
  if (!alertVisible) {
    return null;
  }
  
  return (
    <Alert
      header={t('mfa.alert.header')}
      type="warning"
      dismissible
      onDismiss={handleDismiss}
      action={
        <Button variant="primary" onClick={handleSetupMFA}>
          {t('mfa.alert.button')}
        </Button>
      }
    >
      {t('mfa.alert.message')}
    </Alert>
  );
};
```

### Integration with Account Menu

The MFAAlert component will be integrated into the existing AccountMenuDropdown component:

```jsx
import { MFAAlert } from './MFAAlert';

const AccountMenuDropdown = ({ user, ...props }) => {
  return (
    <div className="account-menu-dropdown">
      <UserInfo user={user} />
      <MFAAlert userType={user.type} />
      <MenuItems {...props} />
    </div>
  );
};
```

## Data Models

### User Type Enum

```typescript
enum UserType {
  IAM = 'IAM',
  ROOT = 'ROOT',
  ROLE = 'ROLE'
}
```

### MFA Status Cache

```typescript
interface MFAStatusCache {
  mfaEnabled: boolean;
  timestamp: number;
}
```

## Error Handling

1. **API Errors**: If the IAM:listMFADevices API call fails, we'll log the error and default to not showing the alert to avoid false positives.

2. **Feature Flag Errors**: If there's an error checking the feature flag, we'll default to not showing the alert.

3. **localStorage Errors**: In environments where localStorage is not available (e.g., private browsing), we'll catch the exceptions and proceed as if the alert hasn't been dismissed.

## Testing Strategy

### Unit Tests

1. Test if the URL construction for the "Setup MFA" button works correctly
2. Test if the component renders the text as expected
3. Test the localStorage persistence logic
4. Test the alert visibility logic based on MFA status

### Integration Tests

1. Test if the alert is visible in the account menu when MFA is not enabled
2. Test if clicking the "Setup MFA" button redirects to the correct page
3. Test if dismissing the alert unmounts the component from the Account Menu
4. Test if the alert remains dismissed after a page reload
5. Mock the IAM:listMFADevices API response to test both MFA enabled and disabled scenarios

### Manual Testing

1. Use an IAM user who has access to call IAM:listMFADevices
2. Verify the alert appears when MFA is not enabled
3. Verify the alert doesn't appear when MFA is enabled
4. Test the dismiss functionality and persistence

## Localization

The following text strings will need to be localized:

1. Alert header: "Your account isn't using MFA"
2. Alert message: "Multi-factor authentication adds an extra layer of security to your account."
3. Button text: "Setup MFA"

These strings will be added to the existing Nav localization setup and translated through Totoro.

## Metrics

We'll implement the following metrics:

1. **MFA_API_LATENCY**: Latency of the IAM:listMFADevices API call
2. **MFA_ALERT_IMPRESSION**: Number of unique users who see the MFA alert
3. **MFA_SETUP_BUTTON_CLICK**: Number of users who click the "Setup MFA" button
4. **MFA_ALERT_DISMISS**: Number of users who dismiss the alert

These metrics will be implemented using the existing metrics instrumentation system in the Navigation component.

## Performance Considerations

To ensure the feature doesn't negatively impact page load performance:

1. **Caching**: Cache the MFA status with a TTL of 1 day
2. **Selective API Calls**: Only make the API call for IAM users and root users
3. **Minimal Processing**: Only store whether MFA is enabled (boolean)
4. **Asynchronous Loading**: Load the MFA status asynchronously after the main navigation components are rendered

## Rollout Strategy

The feature will be deployed behind a feature flag using FMS (Feature Management Service). This allows for:

1. Controlled rollout to different user segments or regions
2. Quick disabling if any issues are discovered
3. A/B testing if needed
4. Metrics collection to validate the impact of the feature
