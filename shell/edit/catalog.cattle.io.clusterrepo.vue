<script>
import CreateEditView from '@shell/mixins/create-edit-view';
import Footer from '@shell/components/form/Footer';
import { LabeledInput } from '@components/Form/LabeledInput';
import { RadioGroup } from '@components/Form/Radio';
import NameNsDescription from '@shell/components/form/NameNsDescription';
import Labels from '@shell/components/form/Labels';
import SelectOrCreateAuthSecret from '@shell/components/form/SelectOrCreateAuthSecret';
import InfoBox from '@shell/components/InfoBox';
import { Checkbox } from '@components/Form/Checkbox';
import { MANAGEMENT, NAMESPACE, CLUSTER_REPO_TYPES } from '@shell/config/types';

export default {
  name: 'CruCatalogRepo',

  emits: ['input'],

  components: {
    Footer,
    RadioGroup,
    LabeledInput,
    NameNsDescription,
    Labels,
    SelectOrCreateAuthSecret,
    InfoBox,
    Checkbox
  },

  mixins: [CreateEditView],

  data() {
    const clusterRepoType = !!this.value.spec.gitRepo ? CLUSTER_REPO_TYPES.GIT_REPO : this.value.isOciType ? CLUSTER_REPO_TYPES.OCI_URL : CLUSTER_REPO_TYPES.HELM_URL;

    return {
      CLUSTER_REPO_TYPES,
      clusterRepoType,
      ociMinWait:    this.value.spec.exponentialBackOffValues?.minWait,
      ociMaxWait:    this.value.spec.exponentialBackOffValues?.maxWait,
      ociMaxRetries: this.value.spec.exponentialBackOffValues?.maxRetries
    };
  },

  computed: {
    inStore() {
      return this.$store.getters['currentProduct']?.inStore || MANAGEMENT;
    },
    secretNamespace() {
      const tryNames = ['cattle-system', 'default'];

      for ( const name of tryNames ) {
        if ( this.$store.getters['cluster/byId'](NAMESPACE, name) ) {
          return name;
        }
      }

      return this.$store.getters[`${ this.inStore }/all`](NAMESPACE)[0]?.id;
    }
  },

  methods: {
    onTargetChange(clusterRepoType) {
      // reset input fields when switching options
      switch (clusterRepoType) {
      case CLUSTER_REPO_TYPES.GIT_REPO:
        this.resetOciValues();
        this.resetHelmValues();
        break;
      case CLUSTER_REPO_TYPES.OCI_URL:
        // set insecurePlainHttp to false as a secondary flag, alongside checking for 'oci://' in the URL, to determine OCI type later
        this.value.spec['insecurePlainHttp'] = false;
        this.resetGitRepoValues();
        this.resetHelmValues();
        break;
      case CLUSTER_REPO_TYPES.HELM_URL:
        this.resetOciValues();
        this.resetGitRepoValues();
        break;
      }
      this.resetClientSecret();
    },
    updateExponentialBackOffValues(key, newVal) {
      if (!Object.prototype.hasOwnProperty.call(this.value.spec, 'exponentialBackOffValues')) {
        this.value.spec['exponentialBackOffValues'] = {};
      }
      // when user removes the value we remove the key too, backend will set the default value
      if (newVal === '') {
        delete this.value.spec.exponentialBackOffValues[key];

        return;
      }

      this.value.spec.exponentialBackOffValues[key] = Number(newVal);
    },
    resetGitRepoValues() {
      delete this.value.spec['gitRepo'];
      delete this.value.spec['gitBranch'];
    },
    resetOciValues() {
      delete this.value.spec['url'];
      delete this.value.spec['insecurePlainHttp'];
      delete this.value.spec['insecureSkipTLSVerify'];
      delete this.value.spec['caBundle'];
      delete this.value.spec['exponentialBackOffValues'];
      this.ociMinWait = undefined;
      this.ociMaxWait = undefined;
      this.ociMaxRetries = undefined;
    },
    resetHelmValues() {
      delete this.value.spec['url'];
    },
    resetClientSecret() {
      this.value.spec['clientSecret'] = null;
    }
  },
};
</script>

<template>
  <form>
    <NameNsDescription
      :value="value"
      :mode="mode"
      :namespaced="isNamespaced"
      @update:value="$emit('input', $event)"
    />

    <h2>{{ t('catalog.repo.target.label') }}</h2>
    <div class="row mb-10">
      <div class="col span-8">
        <RadioGroup
          v-model:value="clusterRepoType"
          :name="clusterRepoType"
          :options="[CLUSTER_REPO_TYPES.HELM_URL, CLUSTER_REPO_TYPES.GIT_REPO, CLUSTER_REPO_TYPES.OCI_URL]"
          :labels="[t('catalog.repo.target.http'), t('catalog.repo.target.git'), t('catalog.repo.target.oci', null, true)]"
          :mode="mode"
          data-testid="clusterrepo-radio-input"
          @update:value="onTargetChange"
        />
      </div>
    </div>

    <div
      v-if="clusterRepoType === CLUSTER_REPO_TYPES.GIT_REPO"
      class="row mb-10"
    >
      <div class="col span-6">
        <LabeledInput
          v-model:value.trim="value.spec.gitRepo"
          :required="true"
          :label="t('catalog.repo.gitRepo.label')"
          :placeholder="t('catalog.repo.gitRepo.placeholder', null, true)"
          :mode="mode"
          data-testid="clusterrepo-git-repo-input"
        />
      </div>
      <div class="col span-6">
        <LabeledInput
          v-model:value.trim="value.spec.gitBranch"
          :sub-label="!value.spec.gitBranch ? t('catalog.repo.gitBranch.defaultMessage', null, true) : undefined"
          :label="t('catalog.repo.gitBranch.label')"
          :placeholder="t('catalog.repo.gitBranch.placeholder', null, true)"
          :mode="mode"
          data-testid="clusterrepo-git-branch-input"
        />
      </div>
    </div>

    <div v-else-if="clusterRepoType === CLUSTER_REPO_TYPES.OCI_URL">
      <InfoBox class="p-10">
        {{ t('catalog.repo.oci.info', null, true) }}
      </InfoBox>
      <LabeledInput
        v-model:value.trim="value.spec.url"
        :required="true"
        :label="t('catalog.repo.oci.urlLabel')"
        :placeholder="t('catalog.repo.oci.placeholder', null, true)"
        :mode="mode"
        data-testid="clusterrepo-oci-url-input"
      />
    </div>

    <LabeledInput
      v-else
      v-model:value.trim="value.spec.url"
      :required="true"
      :label="t('catalog.repo.url.label')"
      :placeholder="t('catalog.repo.url.placeholder', null, true)"
      :mode="mode"
      data-testid="clusterrepo-helm-url-input"
    />
    <SelectOrCreateAuthSecret
      v-model:value="value.spec.clientSecret"
      :mode="mode"
      data-testid="clusterrepo-auth-secret"
      :register-before-hook="registerBeforeHook"
      :namespace="secretNamespace"
      :limit-to-namespace="false"
      :in-store="inStore"
      :allow-ssh="clusterRepoType !== CLUSTER_REPO_TYPES.OCI_URL"
      generate-name="clusterrepo-auth-"
      :cache-secrets="true"
    />

    <div v-if="clusterRepoType === CLUSTER_REPO_TYPES.OCI_URL">
      <div class="row">
        <div class="col span-6">
          <LabeledInput
            v-model:value="value.spec.caBundle"
            class="mt-20"
            type="multiline"
            label="CA Cert Bundle"
            :mode="mode"
            data-testid="clusterrepo-oci-cabundle-input"
          />
          <Checkbox
            v-model:value="value.spec.insecureSkipTLSVerify"
            class="mt-10"
            :mode="mode"
            :label="t('catalog.repo.oci.skipTlsVerifications')"
            data-testid="clusterrepo-oci-skip-tls-checkbox"
          />
          <Checkbox
            v-model:value="value.spec.insecurePlainHttp"
            class="mt-10"
            :mode="mode"
            :label="t('catalog.repo.oci.insecurePlainHttp')"
            data-testid="clusterrepo-oci-insecure-plain-http"
          />
        </div>
      </div>
      <h4 class="mb-10 mt-20">
        {{ t('catalog.repo.oci.exponentialBackOff.label') }}
      </h4>
      <div class="row mb-10 mt-10">
        <div class="col span-4">
          <LabeledInput
            v-model:value.trim="ociMinWait"
            :label="t('catalog.repo.oci.exponentialBackOff.minWait.label')"
            :placeholder="!ociMinWait ? t('catalog.repo.oci.exponentialBackOff.minWait.placeholder') : undefined"
            :mode="mode"
            type="number"
            min="1"
            data-testid="clusterrepo-oci-min-wait-input"
            @update:value="updateExponentialBackOffValues('minWait', $event)"
          />
        </div>
        <div class="col span-4">
          <LabeledInput
            v-model:value.trim="ociMaxWait"
            :label="t('catalog.repo.oci.exponentialBackOff.maxWait.label')"
            :placeholder="!ociMaxWait ? t('catalog.repo.oci.exponentialBackOff.maxWait.placeholder') : undefined"
            :mode="mode"
            type="number"
            min="1"
            data-testid="clusterrepo-oci-max-wait-input"
            @update:value="updateExponentialBackOffValues('maxWait', $event)"
          />
        </div>
        <div class="col span-4">
          <LabeledInput
            v-model:value.trim="ociMaxRetries"
            :label="t('catalog.repo.oci.exponentialBackOff.maxRetries.label')"
            :placeholder="!ociMaxRetries ? t('catalog.repo.oci.exponentialBackOff.maxRetries.placeholder') : undefined"
            :mode="mode"
            type="number"
            min="0"
            data-testid="clusterrepo-oci-max-retries-input"
            @update:value="updateExponentialBackOffValues('maxRetries', $event)"
          />
        </div>
      </div>
    </div>

    <Labels
      default-section-class="mt-20"
      :value="value"
      :mode="mode"
      :display-side-by-side="false"
    />

    <Footer
      data-testid="clusterrepo-footer"
      :mode="mode"
      :errors="errors"
      @save="save"
      @done="done"
    />
  </form>
</template>
