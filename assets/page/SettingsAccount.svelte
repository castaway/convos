<script>
import ChatHeader from '../components/ChatHeader.svelte';
import Button from '../components/form/Button.svelte';
import Checkbox from '../components/form/Checkbox.svelte';
import Icon from '../components/Icon.svelte';
import OperationStatus from '../components/OperationStatus.svelte';
import SelectField from '../components/form/SelectField.svelte';
import TextField from '../components/form/TextField.svelte';
import {createForm} from '../store/form';
import {getContext, onDestroy} from 'svelte';
import {i18n, l, lmd} from '../store/I18N.js';
import {notify} from '../js/Notify';
import {route} from '../store/Route.js';
import {str2array} from '../js/util';

export const title = 'Account';

const api = getContext('api');
const form = createForm();
const registerProtocolHandlerSupported = 'registerProtocolHandler' in navigator;
const themeManager = getContext('themeManager');
const user = getContext('user');
const updateUserOp = api('updateUser');

$: colorSchemeReadonly = $themeManager.hasColorScheme($form.activeTheme) ? false : true;
$: i18n.load($form.lang);
$: if ($form.handle_protocol_irc) registerProtocol('irc');
$: if ($form.handle_protocol_ircs) registerProtocol('ircs');

form.set({
  activeTheme: themeManager.activeTheme,
  colorScheme: themeManager.colorScheme,
  compactDisplay: themeManager.compactDisplay,
  email: user.email,
  expandUrlToMedia: user.expandUrlToMedia,
  highlightKeywords: user.highlightKeywords.join(', '),
  ignoreStatuses: user.ignoreStatuses,
  lang: i18n.lang,
  wantNotifications: notify.wantNotifications,
});

// Using onDestroy() to unsubscribe to notify "update" event
onDestroy(notify.on('update', (notify, changed) => {
  if (!changed.wantNotifications && !changed.desktopAccess) return;
  if (notify.wantNotifications) notify.show($l('You have enabled notifications.'));
}));

function registerProtocol(proto) {
  navigator.registerProtocolHandler(proto, route.urlFor('/register?uri=%s'), $l('Convos %1 handler', proto));
}

function saveAccount() {
  const passwords = [$form.password, $form.password_again];
  if (passwords.join('').length && passwords[0] != passwords[1]) return updateUserOp.error('Passwords does not match.');

  if ($form.wantNotifications) notify.requestDesktopAccess();
  notify.update(form.get(['wantNotifications']));

  if (colorSchemeReadonly) form.set({colorScheme: 'auto'});
  themeManager.update(form.get(['activeTheme', 'colorScheme', 'compactDisplay']));

  const highlightKeywords = str2array(form.get('highlightKeywords'));
  const body = {email: user.email, highlight_keywords: highlightKeywords};
  if (passwords[0]) body.password = passwords[0];
  user.update(form.get(['expandUrlToMedia', 'ignoreStatuses'])).update({highlightKeywords});
  updateUserOp.perform(body);
}
</script>

<ChatHeader>
  <h1>{$l('Account')}</h1>
</ChatHeader>

<main class="main">
  <form method="post" on:submit|preventDefault="{saveAccount}">
    <TextField type="email" name="email" form="{form}" readonly>
      <span slot="label">{$l('Email')}</span>
    </TextField>
    <TextField name="highlightKeywords" form="{form}" placeholder="{$l('whatever, keywords')}">
      <span slot="label">{$l('Notification keywords')}</span>
    </TextField>
    <Checkbox name="wantNotifications" form="{form}">
      <span slot="label">{$l('Enable notifications')}</span>
    </Checkbox>
    <Checkbox name="ignoreStatuses" form="{form}">
      <span slot="label">{$l('Ignore join/part messages')}</span>
    </Checkbox>
    <Checkbox name="expandUrlToMedia" form="{form}">
      <span slot="label">{$l('Expand URL to media')}</span>
    </Checkbox>
    <SelectField name="activeTheme" form="{form}" options="{themeManager.themeOptions}">
      <span slot="label">{$l('Theme')}</span>
    </SelectField>
    <SelectField name="colorScheme" form="{form}" readonly="{colorSchemeReadonly}" options="{themeManager.colorSchemeOptions}">
      <span slot="label">{$l('Color scheme')}</span>
    </SelectField>
    <Checkbox name="compactDisplay" form="{form}">
      <span slot="label">{$l('Enable compact message display')}</span>
    </Checkbox>
    <SelectField name="lang" form="{form}" options="{$i18n.languageOptions}">
      <span slot="label">{$l('Language')} <Icon name="globe"/></span>
    </SelectField>
    <TextField type="password" name="password" form="{form}" autocomplete="new-password">
      <span slot="label">{$l('Password')}</span>
    </TextField>
    <TextField type="password" name="password_again" form="{form}" autocomplete="new-password">
      <span slot="label">{$l('Repeat password')}</span>
    </TextField>

    <p>{$l('Leave the password fields empty to keep the current password.')}</p>

    <div class="form-actions">
      <Button icon="save" op="{updateUserOp}"><span>{$l('Save settings')}</span></Button>
    </div>

    <OperationStatus op="{updateUserOp}"/>
  </form>

  {#if registerProtocolHandlerSupported}
    <form on:submit|preventDefault>
      <h2>Protocol handlers</h2>
      <p>
        {$l('Mark the protocols below that you want %1 to handle, and a popup should ask you for confirmation.', $l('Convos'))}
        {$l('If you do not see any popup, then it probably means %1 already handles the protocol.', $l('Convos'))}
      </p>

      <Checkbox name="handle_protocol_irc" form="{form}">
        <span slot="label">{$l('irc://')}</span>
      </Checkbox>
      <p class="help">{@html $lmd('Test it by clicking on %1.', 'irc://localhost:6667')}</p>

      <Checkbox name="handle_protocol_ircs" form="{form}">
        <span slot="label">{$l('ircs://')}</span>
      </Checkbox>
      <p class="help">{@html $lmd('Test it by clicking on %1.', 'ircs://irc.libera.chat:6697/%23convos')}</p>
    </form>
  {/if}
</main>
